
# Council Election

## Overview

This specification describes how to implement the council election system of the Sparta test network.


## Notation and Types

Variables, parameters and pseudocode are `monospaced`, types are _italicized_. Invocation of `reject` in transaction processing code should be understood to mean that transaction is rejected as invalid.

## Transaction Replay protection

Replay protection mechanism is not specified in this document.

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
typedef set<Seat> Council;

struct Seat {
  Address member, // member filling the seat - unique key for set
  set<Backer> backers;
  uint32_t stake, // council member's stake
};

struct Backer {
  Address member, // unique key for set
  uint32_t stake // backer's stake
};

// This type is needed to keep track of used council stake, and backing stake
// during an election.
struct Stake {
  uint32_t refundable,
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
```

### Commit/Reveal Secret Voting
```c++
struct SecretVote {
  Address candidate,
  byte[32] salt,  // random secret
  uint32_t electionRound
};

typedef byte[32] Commitment; // SHA256 HASH of a SecretVote

struct Vote {
  Address voter,
  Commitment  commitment, // serves as a commitment-id
  Stake stake,
  optional<SecretVote> vote // set when voter reveals their vote
};

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

### `CouncilMinimumStake`
Of type _uint32_, represents the minimum amount council candidate must stake up to enter the applicant pool.

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

### `votes`
Of type _Votes_, represents the votes in the current election round.

### `availableBackingStakes`
Of type _map<Address, uint32_t>_ represents the available backing stakes for each backer. Initialized once when voting period begins.
It is safe to compute it only once provided we guarantee that the council is not mutated during an election.

### `availableCouncilStakes`
Of type _map<Address, uint32_t>_ represents the available council stakes for each council member. Initialized once when voting period begins.
It is safe to compute it only once provided we guarantee that the council is not mutated during an election.


## Events
These can be as a result of an incoming transaction, a method invocation from another module, or a chain event.

### `tick(uint32_t blockHeight)`
trigger: external call

Invoked on every block interval when an election is in progress.
electionStarted() must have been called once.

```c++
  if (blockHeight != periodEndsAtBlockHeight) return;

  // trigger internal events
  if (period == ANNOUNCING) {
    announcePeriodEnded();
  } else if (period == VOTING) {
    votingPeriodEnded();
  } else if (period == REVEALING) {
    revealingPeriodEnded();
  }
```

### `electionStarted(uint32_t blockHeight)`
trigger: external call

Marks the start of an election.
Should be triggered when the Council resigns, it's term ends, or an early election proposal passes.

Must only be called once during the lifetime of an election.

#### processing
```c++
assert(period == NULL);
period = ANNOUNCING;
announcePeriodStarted(blockHeight);
electionRound = 0; // initialize round
initializeBackingStakes()
initializeCouncilStakes()
```

### `announcePeriodStarted(uint32_t blockHeight)`
trigger: internal call

Invoked when a new round of the election process starts and the announcing period begins.

#### Processing

```c++
// Restarting election
electionRound++;
periodEndsAtBlockHeight = blockHeight + LengthOfAnnouncePeriod;
```

### `announcePeriodEnded()`
trigger: internal call

Triggered when chain reaches height `announcingPeriodEndsAtBlockHeight`

#### processing
```c++
if (applicantsPool.size < CouncilSize) {
  // start new election round - continue with same applicantsPool
  announcePeriodStarted(periodEndsAtBlockHeight);
} else {
  votingPeriodStarted(periodEndsAtBlockHeight);
}
```

### `announceCandidacy(Address applicant, uint32 stake)`
trigger: transaction

Triggered when a member announces their candidacy by staking `stake` amount of tokens, or is adding additional stake to back their candidacy.

```c++
assert(isMember(applicant))
assert(tx.sender == applicant) // only applicant can provide council stake
assert(period == ANNOUNCING)

Stake applicantStake = applicantsPool[applicant]
uint32_t availableCouncilStake = availableCouncilStakes[applicant]

if (availableCouncilStake > 0) {
  // Existing council member can transfer their own council stake
  if (availableCouncilStake >= stake) {
    availableCouncilStake -= stake
    applicantStake.transferred += stake
  } else {
    uint32_t diff = stake - availableCouncilStake
    assert(Balances[applicant] >= diff)
    applicantStake.transferred += availableCouncilStake
    availableCouncilStake = 0
    applicantStake.refundable += diff
    Balances[applicant] -= diff
  }
  availableCouncilStakes[applicant] = availableCouncilStake
} else {
  assert(Balances[applicant] >= stake)
  applicantStake.refundable += stake
  Balances[applicant] -= stake
}

// Must meet minimum stake requirement
assert(applicantStake.getTotal() < CouncilMinimumStake)

applicantsPool.set(applicant, applicantStake)  
```

### `withdrawCandidacy(Address candidate)`
trigger: transaction, internal call

Triggered when a member which has announced their candidacy is withdrawing it.

```c++
assert(period == ANNOUNCING)
stake = applicantsPool[candidate]
// return stake to applicant - should probably delay till end of election
// but mark candidacy withdrawn so not to count votes
Balances[candidate] += stake.refundable

// If applicant is a council member return any transferred stake
if (stake.transferred > 0) {
    councilStakes[applicant] += stake.transferred
}

applicantsPool.remove(candidate)
```

### `votingPeriodStarted(uint32_t blockHeight)`
trigger: internal call

```c++
assert(period == ANNOUNCING)
assert(applicantsPool.size >= CouncilSize)
candidatePool = sortByTotalStakeDecending(applicantsPool).slice(0, CouncilCandidacyLimit)
period = VOTING;
periodEndsAtBlockHeight = blockHeight + LengthOfVotingPeriod
```

### `secretVoteSubmitted(Address voter, Commitment commitment, uint32_t stake)`
trigger: transaction

Triggered when a member submits a voting transaction.

```c++
assert(period == VOTING)
assert(isMember(voter))
assert(!Votes.has(commitment))
assert(availableBackingStakes[voter] + Balance[voter] > stake)
// If transferable amount available use it
// Create a new Stake
// Update availableBackingStakes[voter]
// Update voter balance
// Create new Vote, insert into Votes
```

### `votingPeriodEnded()`
trigger: internal call

```c++
assert(period == VOTING)
revealPeriodStarted(periodEndsAtBlockHeight)
```


### `revealPeriodStarted(uint32_t blockHeight)`
```c++
assert(period == VOTING)
period = REVEALING
periodEndsAtBlockHeight = blockHeight + LengthOfRevealingPeriod
```
### `revealSubmitted(Address voter, Commitment commitment, SecretVote vote)`
trigger: transaction

```c++
assert(tx.sender == voter)
assert(Votes.has(commitment))
assert(sha256(vote) == commitment)
assert(candidatePool.has(vote.candidate))
Votes[commitment].vote = vote
```

### `revealPeriodEnded()`
```
Tally votes => create a new Council
(what is the success criteria for a valid/successful election:
  minimum backing stake for each candidate,
  minimum total backing stake == %participation/quorum,
  size of elected council < CouncilSize)

refund vote stakes of un-revealed commitments
if (successfulElection) {
  refund all vote stakes for candidates that did not get elected
  refund all unused availableBackingStakes
  refund all applicantsPool stakes for applicants that did not get elected
  refund all unused availableCouncilStakes
  set new council
  electionCompleted()
} else {
  refund refundable vote stakes
  return transferred stake from votes back to backingStakes
  clear Votes
  clear candidatePool
  start new election round - startRevealingPeriod()
}
```

### `electionCompleted()`
```
period = NULL
```
