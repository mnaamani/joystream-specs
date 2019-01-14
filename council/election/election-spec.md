
# Council Election

## Overview

This specification describes how to implement the council election system of the Sparta test network.


## Notation and Types

Variables, parameters and pseudocode are `monospaced`, types are _italicized_. Invocation of `reject` in transaction processing code should be understood to mean that transaction is rejected as invalid.

### CouncilElectionPeriod

```c++
enum {
  NULL = 0, // initial state
  ANNOUNCING = 1,
  VOTING = 2,
  REVEALING = 3,
}
```

### Council
```c++
struct Council {
  set<Seat> seats;
  set<Backer> backers;
}

struct Seat {
  Address councilMember, // member filling the seat - unique key for set
  set<Address>  backers,
  Stake stake, // council member's stake
};

struct Backer {
  Address member,
  Stake stake
};

// total stake = refundable + transferred
// This type is needed to keep track of used council stake, and backing stake
// during an election.
struct Stake {
  uint32_t refundable
  uint32_t transferred
};
```

### CouncilApplicantsPool

```c++
typedef map<Address applicant, Stake stake> CouncilApplicantsPool;
```

### CouncilCandidatePool
```c++
typedef Address[CouncilCandidacyLimit] CouncilCandidatePool;
This can be computed why store it as state?
```

### Commit/Reveal Secret Voting
```c++
struct SecretVote {
  Address candidate,
  uint32_t stake,
  byte[32] salt  // random secret - revealed during reveal period
};

typedef byte[32] Commitment; // SHA256 HASH of a SecretVote

struct Vote {
  Address voter,
  Commitment  commitment, // serves as a commitment-id
  optional<SecretVote> vote // updated when vote is revealed
};

struct Voter {
  Address voter,
  Stake stake,
  set<Vote> votes
}

typedef set<Voter> Voters;
typedef set<Vote> Votes;
```

## Parameters hardcoded in runtime

### `LengthOfAnnouncePeriod`

Of type _uint32_, represents length of time in blocks of announcing period.

### `LengthOfVotingPeriod`

Of type _uint32_, represents length of time in blocks of voting period.

### `LengthOfRevealingPeriod`

Of type _uint32_, represents length of time in blocks of revealing period.

## Dependency on other modules

### `Council`

### `Balances`


## System Governance Parameters

### `CouncilSize`
Of type _uint32_, represents the number of seats that make up the council.

### `CouncilStakingLimit`
Of type _uint32_, represents the minimum amount of stake a council candidate must put up directly to enter the candidate pool.

### `CouncilCandidacyLimit`
Of type _uint32_t, represents the maximum size of the candidate pool.

## State

### `period`

Of type _[CouncilElectionPeriod](#CouncilElectionPeriod)_, represents the current stage of the council election process.

### `periodEndsAtBlockHeight`

Of type _uint32_, represents the block at which the current period will end.

### `applicantsPool`

Of type _CouncilApplicantsPool_, represents all council candidates that have applied and staked tokens during the announcing period.

### `electionRound`
Of type _uint32_ , represents the round of the current elections.

### `Voters`
### `Votes`

## Events
These can be as a result of an incoming transaction, a method invocation from another module, or a chain event.

### `tick(uint32_t blockHeight)`
trigger: external call

Invoked on every block interval when an election is in progress.
electionStarted() must have been called once already.

```c++
  if (blockHeight != periodEndsAtBlockHeight) return

  // trigger internal events
  if (period == ANNOUNCING) {
    announcePeriodEnded()
  } else if (period == VOTING) {
    votingPeriodEnded()
  } else if (period == REVEALING) {
    revealingPeriodEnded()
  }
```

### `electionStarted(uint32_t blockHeight)`
trigger: external call

Marks the start of an election.
Should be triggered when the Council resigns or it's term ends.
Must only be called once during the lifetime of an election.

#### processing
```c++
assert(period == NULL)
announcePeriodStarted(blockHeight)
electionRound = 0 // first round
```

### `announcePeriodStarted(uint32_t blockHeight)`
trigger: internal call

Invoked when a new round of the election process starts and the announcing period begins.

#### Processing

```c++
// Restarting election
electionRound++
period = ANNOUNCING
periodEndsAtBlockHeight = blockHeight + LengthOfAnnouncePeriod
```

### `announcePeriodEnded()`
trigger: internal call

Triggered when chain reaches height `announcingPeriodEndsAtBlockHeight`

#### processing
```c++
if (applicantsPool.size < CouncilSize) {
  // start new election round
  announcePeriodStarted(periodEndsAtBlockHeight)
} else {
  votingPeriodStarted(periodEndsAtBlockHeight)
}
```

### `announceCandidacy(Address applicant, uint32 stake)`
trigger: transaction

Triggered when a member announces their candidacy by staking `stake` amount of tokens, or is adding additional stake to back their candidacy.

```c++
assert(isMember(applicant))
assert(Balances[applicant] >= stake)
assert(period == ANNOUNCING)

applicantStake; // 0 stake

// if existing applicant - add more stake
if applicantsPool.has(applicant) {
  applicantStake = applicantsPool.get(applicant)
  applicantStake.refundable += stake
  applicantStake.total += stake
} else {
  // first time applicant can transfer stake once
  if CurrentCouncil.has(applicant) {
    applicantStake.total = CurrentCouncil.get(applicant).stake
  }
  applicantStake.total += stake
  // only non-transferred stake is refundable
  applicantStake.refundable += stake

  // Must meet minimum stake requirement
  if(applicantStake.total < CouncilStakingLimit) abort()
}

applicantsPool.set(applicant, applicantStake)
Balances[applicant] -= stake
```

### `withdrawCandidacy(Address candidate)`
trigger: transaction

Triggered when a member which has announced their candidacy is withdrawing it.

```c++
assert(period != NULL)
stake = applicantsPool.get(candidate)
// return stake to applicant - should probably delay till end of election
// but mark candidacy withdrawn so not to count votes
Balances[candidate] += stake.refundableStake
applicantsPool.remove(candidate)
```

### `votingPeriodStarted(uint32_t blockHeight)`
trigger: internal call

```c++
assert(period == ANNOUNCING)
assert(candidatePool.size >= CouncilSize)
period = VOTING
periodEndsAtBlockHeight = blockHeight + LengthOfVotingPeriod
```
### `voteCommitmentSubmitted(VoteCommitment commitment)`
trigger: transaction

Trigerred when a member submits a voting transaction.

```c++
assert(period = VOTING)
assert(isMember(commitment.voter))
assert(Balances[commitment.voter] >= commitment.)
```
### `voteCommitmentWithdrawn(uint32_t commitmentId)`

### `votingPeriodEnded()`
trigger: internal call

```c++
assert(period == VOTING)
period = REVEALING
periodEndsAtBlockHeight = blockHeight + LengthOfRevealingPeriod
```


### `revealPeriodStarted()`
### `revealSubmitted(Reveal reveal)`
### `revealPeriodEnded()`
### `electionCompleted()`
