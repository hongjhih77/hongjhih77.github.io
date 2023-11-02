---
title:  "Distributed Transaction"
date:   2023-10-27 00:00:00 +0800
toc: true
toc_label: "Table of Contents"
toc_sticky: true
---

## Distributed Transactions

> The first option could be to just not split the data apart in the first place.

Aggregate the infos about Distributed Transactions across multiple references.

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

Transfer $30 from A to B across banks.

If A fails to deduct due to insufficient balance, then the transfer will directly fail and return an error; 
if the deduction is successful, then the next transfer operation TransIn will be carried out, because TransIn does not have the problem of insufficient balance, 
and it can be assumed that the transfer operation will definitely succeed.

![2PC_ex_n.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_ex_n.png)

Crash after commit
![2PC_ex_crash_after_commit.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_ex_crash_after_commit.png)

> [My Note] Idempotence must be applied when retrying.

Crash before commit
![2PC_ex_crash_before_commit.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/2PC_ex_crash_before_commit.png)

---
### SAGA
- Coordinate multiple changes in state.
- Avoids the need for locking resources.
- Break down these LLTs(long lived transactions) into a sequence of transactions, each of which can be handled independently.
- Have atomicity for each individual transaction inside the overall saga

#### Saga Failure Modes

- **Backward recovery**: A **compensating actions** that allow us to undo previously committed transactions.
- **Forward recovery**: pick up from the point where the failure occurred, **retry** transactions.
- A saga allows us to recover from business failures, not technical failures.
- The saga assumes the underlying components are working properly—that the underlying system is reliable, and that we are then coordinating the work of reliable components.

##### Saga rollbacks
![Saga_rollback.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/Saga_rollback.png)

Reordering workflow steps to reduce rollbacks

![Saga_reorder_deduce_rollback.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/Saga_reorder_deduce_rollback.png)

#### Implementing Sagas

##### Orchestrated sagas
- A central coordinator.
- A command-and-control approach: the orchestrator controls what happens and when
- A good degree of visibility

![Saga_orch.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/Saga_orch.png)

Cons:
- Domain coupling.
- Logic that should otherwise be pushed into the services can start to become absorbed in the orchestrator instead.

##### Choreographed sagas
- Trust-but-verify architecture.
- Heavy use of events: events are broadcast in the system, and interested parties are able to receive them.
- Use some sort of message broker to manage the reliable broadcast.
- Less coupled.

![Saga_chor.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction/Saga_chor.png)

Cons:
- Harder to work out what is going on.
- Lack a way of knowing what state a saga

Fix cons.:
- A **correlation ID**, we can put it into all of the events that are emitted as part of this saga.
- Have a service whose job it is to just vacuum up all these events and present a view.

### [Diagrams from DTM tutorial](https://en.dtm.pub/practice/saga.html#split-into-subtransactions)

![DTM_saga.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction%2FDTM_saga.png)

Failure rollback
![DTM_saga_failed.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction%2FDTM_saga_failed.png)

---
### [XA](https://en.dtm.pub/practice/xa.html#what-is-xa)

XA is a specification for distributed transactions proposed by the X/Open organization. The X/Open Distributed Transaction Processing (DTP) model envisages three software components:

- An application program (AP) defines transaction boundaries and specifies actions that constitute a transaction.
- Resource managers (RMs, such as databases or file access systems) provide access to shared resources.
- A separate component called a transaction manager (TM) assigns identifiers to transactions, monitors their progress, and takes responsibility for transaction completion and for failure recovery.

![XA.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction%2FXA.png){: .align-center}

XA is divided into two phases.

- Phase 1 (prepare): All participating RMs prepare to execute their transactions and lock the required resources. When each participant is ready, it report to TM.

- Phase 2 (commit/rollback): When the transaction manager (TM) receives that all participants (RM) are ready, it sends commit commands to all participants. Otherwise, it sends rollback commands to all participants.

**At present, almost all popular databases support XA transactions, including Mysql, Oracle, SqlServer, and Postgres**

#### XA in Mysql [5]
``` sql
XA start '4fPqCNTYeSG' -- start a xa transaction
UPDATE `user_account` SET `balance`=balance + 30,`update_time`='2021-06-09 11:50:42.438' WHERE user_id = 1
XA end '4fPqCNTYeSG'
-- if connection closed before `prepare`, then the transaction is rolled back automatically
XA prepare '4fPqCNTYeSG'

-- When all participants have all prepared, call commit in phase 2
xa commit '4fPqCNTYeSG'

-- When any participants have failed to prepare, call rollback in phase 2
-- xa rollback '4fPqCNTYeSG'
```

#### Business Scenario
A needs to transfer money across a bank to B.

A successful transaction:
![XA_normal.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction%2FXA_normal.png)

The timing diagram for failure is as follows:
![XA_failed.png](https://hongjhih77.github.io/img/2023-10-27-Distributed%20Transaction%2FXA_failed.png)

---
### Others
  - Three-Phase Commit (Propose, Prepare, Commit)
  - TCC (Try, Confirm, Cancel)
  - Distributed Transactions with Calvin
  - Distributed Transactions with Spanner
  - Distributed Transactions with Percolator

---
### Reference
- [1] [Building Microservices, 2nd Edition -- Ch.6 Workload](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/)
- [2] [Database Internals -- Ch13 Distributed Transactions](https://learning.oreilly.com/library/view/database-internals/9781492040330/)
- [3] [THE LIMITS OF THE SAGA PATTERN](https://www.ufried.com/blog/limits_of_saga_pattern)
- [4] [Distributed Transactions & Two-phase Commit by Animesh Gaitonde](https://medium.com/geekculture/distributed-transactions-two-phase-commit-c82752d69324)
- [5] DTM : DTM is a distributed transaction framework by GoLang
  - <https://github.com/dtm-labs/dtm>
  - <https://en.dtm.pub/guide/start.html>
- [5] Another tool like [seata](https://github.com/seata/seata) by java