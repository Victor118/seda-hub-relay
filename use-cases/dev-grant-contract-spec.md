# Use Case: Dev Grants Paid on Merge

> Example use case of [seda-hub-relay](../seda-relay-contract-spec.md). Not part of the relay itself.

## Idea

A grant is a contract between a developer and the Cosmos Hub. The developer instantiates a contract that holds the terms: their payout address, their GitHub account, the deliverable, the amount and the deadline. A governance proposal funds it from the community pool. When the developer's pull request is merged, a SEDA oracle program attests it, the relay proves the result on the Hub, and the contract pays the developer.

- **No intermediary holds the funds.** Today, community pool grants usually go to a multisig or a grant body that decides when to pay. Here, nobody can delay, redirect or withhold the payment once the deliverable is merged.
- **Quality control stays with the maintainers.** The trigger is a merge, so the people who review the code decide whether the work is accepted.
- **Governance keeps the last word.** The `x/gov` module is the contract admin: it can cancel, refund or amend through a new proposal.
- **Funds go back automatically.** If nothing is delivered by the deadline, they return to the community pool.

## Flow

```
1. Developer instantiates DevGrant { payee, github_user_id, repo, issue, amount, deadline, admin: gov }
2. Developer submits a proposal: MsgCommunityPoolSpend { recipient: <grant contract>, amount }
   Voters see who is paid, for what, how much, and until when.
3. Proposal passes. Anyone calls Activate { proposal_id }: the contract checks the proposal
   (/cosmos.gov.v1.Query/Proposal) and its own balance.
4. Developer opens a PR, maintainers review and merge it.
5. Developer (or anyone) calls Claim { pr_number }: the contract opens a relay request for the
   GitHub oracle program with inputs { repo, pr_number, issue }.
6. A relayer gets the SEDA result proven on the Hub. The relay calls back the grant contract.
7. The contract checks the attested facts against its terms and pays the payee.
   Past the deadline without delivery: Expire returns the funds to the community pool.
```

## Oracle Program: `github-pr-merged`

Inputs: `{ owner, repo, pr_number, issue_number }`.

It uses the public GitHub REST API without authentication, since an oracle program cannot hold a secret token. The unauthenticated limit is 60 requests per hour per IP, and each executor calls from its own IP.

| Call | Fields used |
|---|---|
| `GET /repos/{owner}/{repo}/pulls/{pr_number}` | `merged`, `merged_at`, `base.ref`, `user.id`, `merge_commit_sha`, `body` |
| `GET /repos/{owner}/{repo}/issues/{issue_number}` | `state`, `state_reason`, `pull_request` (must be absent: it is an issue) |

Output, Borsh or fixed binary encoding (no floats, no free text), identical across executors:

```
{
  merged: bool,
  base_ref: string,
  author_id: u64,
  merged_at: u64,              // unix seconds
  merge_commit_sha: [u8; 20],
  closes_issue: bool,          // body contains a closing keyword for #issue_number
  issue_closed_completed: bool // state == closed && state_reason == completed
}
```

The facts come from GitHub, so the oracle trusts GitHub's API. That is reasonable for a project whose development already lives on GitHub, and should be stated in every grant.

### Linking the PR to the issue

GitHub's reliable link between a PR and the issues it closes (`closingIssuesReferences`) is only available through GraphQL, which requires authentication. The REST issue events do not carry the closing commit for squash merges (checked on cosmos/cosmos-sdk#26673, closed by #26735: the `closed` event has no `commit_id`).

The v1 criterion therefore combines two public facts:

- the PR body contains a closing keyword for the issue (`Closes #N`, `Fixes #N`, `Resolves #N`, case-insensitive);
- the issue is closed as `completed`.

The PR author can edit the body after the merge. Cheating would still require a merged PR into the base branch **and** maintainers closing the issue as completed, and governance can cancel a grant. A stronger variant would read `closingIssuesReferences` through a data proxy that holds a GitHub token.

## Contract

### Instantiate

```rust
#[cw_serde]
pub struct InstantiateMsg {
    /// Paid on delivery. Fixed: it is part of what governance votes on.
    pub payee: String,
    /// Numeric GitHub user id (stable across renames), not the login.
    pub github_user_id: u64,
    pub owner: String,            // "cosmos"
    pub repo: String,             // "gaia"
    pub base_ref: String,         // "main"
    pub issue_number: u64,        // 4023
    pub amount: Coin,             // in uatom
    pub deadline: Timestamp,
    /// seda-hub-relay contract, and the oracle program accepted for this grant.
    pub relay: String,
    pub exec_program_id: String,
    pub tally_program_id: String,
    pub min_replication_factor: u16,
    /// Bounty paid to the relayer for the claim, out of the grant funds.
    pub relay_bounty: Coin,
}
```

The contract admin (wasmd admin and the contract's own `admin` field) is the gov module address. The grant terms are readable on-chain before the vote, through the `Terms` query.

### Execute

```rust
#[cw_serde]
pub enum ExecuteMsg {
    /// Anyone, once the funding proposal has passed.
    Activate { proposal_id: u64 },
    /// Anyone (usually the developer), once the PR is merged.
    Claim { pr_number: u64 },
    /// Called back by the relay contract with the proven result.
    SedaResult { request_id: u64, exit_code: u32, consensus: bool, result: Binary, block_timestamp: u64 },
    /// Anyone, after the deadline, if not paid.
    Expire {},
    /// Gov only: refund to the community pool now, or change the deadline.
    Cancel {},
    ExtendDeadline { deadline: Timestamp },
}
```

### Checks

**`Activate`**

- `/cosmos.gov.v1.Query/Proposal` returns `PROPOSAL_STATUS_PASSED`. Its messages contain a `MsgCommunityPoolSpend` to this contract for at least `amount + relay_bounty`. This query is already in Gaia's accept list since [#4082](https://github.com/cosmos/gaia/pull/4082).
- The contract balance covers `amount + relay_bounty`.

**`Claim`**

- The grant is active, not paid, and the deadline has not passed. No other claim is pending.
- The contract sends `Request` to the relay with the configured oracle program, inputs `{ owner, repo, pr_number, issue_number }`, `callback: true`, and `relay_bounty` attached. The relay imposes the memo, which makes the request unique.

**`SedaResult`**

- `info.sender == relay` and `request_id` is the pending claim.
- `consensus && exit_code == 0`, otherwise the claim is cleared and can be retried.
- The decoded result satisfies every term:
  - `merged` and `base_ref == config.base_ref`;
  - `author_id == config.github_user_id`;
  - `merged_at ≤ deadline`;
  - `closes_issue` and `issue_closed_completed`.
- Then `BankMsg::Send { to: payee, amount }`, and the grant is marked paid with the PR number and merge commit.

A failed check only clears the claim. It does not cancel the grant, because the developer may still deliver another PR before the deadline.

**`Expire` and `Cancel`**

Send the remaining balance back with `DistributionMsg::FundCommunityPool` (`cosmwasm_1_3`), and mark the grant closed.

### Milestones

A larger grant can list several tranches, each with its own `issue_number`, `amount` and `deadline`, and pay them independently. An upfront tranche with no condition is possible if governance accepts it.

## Open Questions

1. **Timing of the merge vs. the deadline.** `merged_at` comes from GitHub and is checked against the deadline. Delivery is therefore judged by the merge date, not by when the claim is relayed.
2. **Reverted PRs.** A merge that is reverted later still counts. Governance can still claw back by proposal, but the contract will have paid already.
3. **Several contributors.** v1 pays a single author. Co-authored work would need a split defined in the terms.
4. **Reproducible build of the oracle program.** Voters must be able to check that `exec_program_id` is the published `github-pr-merged` source (see the relay spec).
