---
tags: [Mermaid]
mermaid: true
---

# 2PC Algorithm

2PC (Two-Phase Commit) is an atomic commitment protocol (ACP).

## References

The algorithm attempts to faithfully implement 2PC as it is described in the
following sources:

- Bernstein, Hadzilacos, and Goodman, Concurrency Control and Recovery in
  Database Systems, 7.4.  This book may be downloaded for free from
  <https://www.microsoft.com/en-us/research/people/philbe/>.

Additional references:

- Kleppmann, Distributed Systems 7.1: Two-phase commit [video]. In YouTube.
  <https://www.youtube.com/watch?v=-_rdWB9hN1c&ab_channel=MartinKleppmann>
- Two-phase commit protocol. In Wikipedia.
  <https://en.wikipedia.org/wiki/Two-phase_commit_protocol>

The following important deviations are present in this implementation:

- The concept of an epoch to commit or abort across a sequence of values.
- All processes start with full knowledge of all other processes, which is
  a deviation from the algorithm as described in Bernstein, Hadzilacos, and
  Goodman.

## The Algorithm

The algorithm is run over a series of epoch. In each epoch, a decision is
reached to either commit a value or abort. After an epoch is complete, the next
epoch begins.

One process takes the role of a coordinator and all other processes take on the
role of a participant.

Prior to the first epoch, each process initializes a local
TwoPhaseCommitContext. This initial context contains the coordinator,
participants, and this_process. this_process contains the value of the local
process. If this_process is equal to the value stored in the coordinator field,
then that process is the coordinator; otherwise, the process is a participant.

To start the epoch, an external component of the coordinator process generates
a Start event, which includes a proposed value. In processing the event, the
algorithm generates actions to send VoteRequest to each participant and set
a timeout. Each participant responds in one of the following ways: 1) with
a VoteResponse(true) message; 2) with a VoteResponse(false) message; or 3) with
no message (the participant fails to respond before the coordinator's timeout
occurs).

The coordinator collects the votes. If all votes are received on time and the
votes are all true, then the coordinator decides to commit. If all votes are
received and at least one of them is false, then the coordinator decides to
abort. If the coordinator doesn't receive all votes before it times out, then
the coordinator decides to abort.

When the coordinator decides to commit, it records its decision locally and
then sends a Commit message to all the other participants.

When the coordinator decides to abort, it records its decision locally and then
sends an Abort message to all the other participants.

Then the next epoch begins.

## States

### Coordinator

<div class="mermaid">
stateDiagram-v2
    direction LR
    [*] --> WaitingForStart
    WaitingForStart --> Voting: Start
    Voting --> Abort: Alarm + Timeout
    Voting --> Voting: Deliver(Vote)
    state decision &lt;&lt;choice&gt;&gt;
    Voting --> decision
    decision --> Abort: Any Participant Vote = No
    decision --> WaitingForVote: All Participant Votes = Yes
    state decision2 &lt;&lt;choice&gt;&gt;
    WaitingForVote --> decision2
    decision2 --> Abort: Coordinator Vote = No
    decision2 --> Commit: Coordinator Vote = Yes
    state join_ack &lt;&lt;join&gt;&gt;
    Abort --> join_ack
    Commit --> join_ack
    state "WaitingForDecisionAck" as wfda
    join_ack --> wfda
    wfda --> wfda: Deliver(DecisionAck)
    wfda --> WaitingForStart: Alarm + Timeout or All Participants Acked
</div>

### Participant

<div class="mermaid">
stateDiagram-v2
    direction LR
    [*] --> WaitingForVoteRequest
    WaitingForVoteRequest --> WaitingForVote: Deliver(VoteRequest)
    WaitingForVote --> Voted: This Participant's Vote = Yes
    WaitingForVote --> Abort: This Participant's Vote = No
    Voted --> Voted: On Alarm + Timeout send DecisionRequest
    Voted --> Abort: Deliver(Abort)
    Voted --> Commit: Deliver(Commit)
    state join_start &lt;&lt;join&gt;&gt;
    Abort --> join_start
    Commit --> join_start
    join_start --> WaitingForVoteRequest: Send DecisionAck
</div>

## Example Sequences

### 2 Processes Committing

The following sequence occurs when two processes commit during the epoch:

<div class="mermaid">
sequenceDiagram
    participant CE as Coordinator Component
    participant C as Coordinator Algorithm
    participant P as Participant Algorithm
    participant PE as Participant Component

    CE->>+C: Start(value)
    rect rgb(192, 192, 192)
    Note over C,P: Vote Phase
    C->>+P: VoteRequest(epoch, value)
    P->>+PE: RequestForVote(value)
    PE-->>-P: Vote(vote=true)
    P-->>-C: VoteResponse(epoch, vote=true)
    end
    rect rgb(192, 192, 192)
    Note over C, P: Commit Phase
    C->>+CE: RequestForVote()
    CE-->>-C: Vote(vote=true)
    C->>P: Commit(epoch)
    C-->>-CE: Commit()
    end
    C->>CE: RequestForStart()
</div>

### 2 Processes with Participant Aborting

The following sequence occurs when the participant aborts during the epoch:

<div class="mermaid">
sequenceDiagram
    participant CE as Coordinator Component
    participant C as Coordinator Algorithm
    participant P as Participant Algorithm
    participant PE as Participant Component

    CE->>+C: Start(value)
    rect rgb(192, 192, 192)
    Note over C,P: Vote Phase
    C->>+P: VoteRequest(epoch, value)
    P->>+PE: RequestForVote(value)
    PE-->>-P: Vote(vote=false)
    P-->>-C: VoteResponse(epoch, vote=false)
    end
    rect rgb(192, 192, 192)
    Note over C, P: Commit Phase
    Note right of P: Abort immediately
    P->>PE: Abort()
    C->>P: Abort(epoch)
    C-->>-CE: Abort()
    end
    C->>CE: RequestForStart()
</div>

### 2 Processes with Coordinator Timeout

The following sequence occurs when the coordinator aborts because the
participant has not responded before a timeout:

<div class="mermaid">
sequenceDiagram
    participant CE as Coordinator Component
    participant C as Coordinator Algorithm
    participant P as Participant Algorithm
    participant PE as Participant Component

    CE->>+C: Start(value)
    rect rgb(192, 192, 192)
    Note over C,P: Vote Phase
    C->>+P: VoteRequest(epoch, value)
    end
    Note right of C: Timeout!
    rect rgb(192, 192, 192)
    Note over C, P: Commit Phase
    C->>P: Abort(epoch)
    P->>PE: Abort()
    C-->>-CE: Abort()
    end
    C->>CE: RequestForStart()
</div>

## Implementation Notes

### Coordinator Commits

In this implementation, the coordinator and participants are all committers.
In some descriptions of the algorithm, such as the one by Kleppmann, the
coordinator and participant roles are distinct and the coordinator is not
a committer.
