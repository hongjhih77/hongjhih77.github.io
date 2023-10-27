---
layout: post
title:  "Distributed Transaction"
date:   2023-10-27 00:00:00 +0800
categories: jekyll update
---

## Distributed Transactions

Aggregate the infos about Distributed Transactions

---
### Two Phase Commit (2PC)
#### Propose(Prepare) phase, Commit/Abort phase
- A leader (or coordinator) that holds the state, collects votes
- Prepare : If a cohort decides that it can commit, it notifies the coordinator about the positive vote.
- Commit/abort : If even one of the cohorts votes to abort the transaction, the coordinator sends the Abort message to all of them.

![2PC_n.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_n.png)

#### Cohort Failures in 2PC
![2PC_cf.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_cf.png)

#### Coordinator Failures in 2PC

One of the cohorts does not receive a commit or abort command from the coordinator during the second phase
![2PC_lf.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_lf.png)

The coordinator fails after collecting the votes, but before broadcasting vote results, the cohorts end up in a state of uncertainty.
![2PC_lf_after_v.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_lf_after_v.png)

#### Cons
- Latency: The more participants you have, and the more latency you have in the system.
- Transaction Coordinator: The Transaction Coordinator becomes a single point of failure at times. The Transaction Coordinator may go down before sending a commit message to all the participants. In such cases, all the transactions running on the participants will go in a blocked state. They would commit only once the coordinator comes up & sends a commit signal.
- Distributed locks [1]:
    - The workers/cohorts need to lock local resources to ensure that the commit can take place during the second phase.
    - The challenges of coordinating locks among multiple participants.

#### Example
Business Scenario example using [DTM](https://en.dtm.pub/practice/msg.html#success-process) as a TM (Transaction Manager)

An inter-bank transfer is a typical distributed transaction scenario, where A needs to transfer money across a bank to B. Suppose that only the transfer from A may fail and the transfer to B is able to eventually succeed

![2PC_ex_n.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_ex_n.png)

#####  Crash after commit
![2PC_ex_crash_after_commit.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_ex_crash_after_commit.png)

> [My Note] Idempotence when retrying need to be applied.

##### Crash before commit
![2PC_ex_crash_before_commit.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_ex_crash_before_commit.png)

---
### Reference
- [1] [Building Microservices, 2nd Edition -- Ch.6 Workload](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/)
- [2] [Database Internals -- Ch13 Distributed Transactions](https://learning.oreilly.com/library/view/database-internals/9781492040330/)
- [3] [THE LIMITS OF THE SAGA PATTERN](https://www.ufried.com/blog/limits_of_saga_pattern)
- [4] [Distributed Transactions & Two-phase Commit by Animesh Gaitonde](https://medium.com/geekculture/distributed-transactions-two-phase-commit-c82752d69324)
- [5] DTM : DTM is a distributed transaction framework by GoLang
  - <https://github.com/dtm-labs/dtm>
  - <https://en.dtm.pub/guide/start.html>
- [5] other tool like [seata](https://github.com/seata/seata) by java