---
title: "From CAP to Raft: The Complete Reasoning Chain Behind Distributed Systems"
date: 2026-07-07
draft: false
tags:
  - Distributed Systems
  - CAP
  - Raft
  - Consensus
  - Kubernetes
categories:
  - Many WHYs
---
# From CAP to Raft: The Complete Reasoning Chain Behind Distributed Systems

Many engineers know:

- CAP theorem
- Raft consensus algorithm
- etcd
- ZooKeeper

But these concepts are often learned separately.

A more important question is:

> Why did these technologies appear?

The evolution of distributed systems follows a natural path:

```

Why do we need distributed systems?
|
v
Why does replication introduce consistency problems?
|
v
Why do we have CAP theorem?
|
v
Why must systems choose between CP and AP?
|
v
How do real systems make this tradeoff?
|
v
Why do we need consensus algorithms?
|
v
Why do we need Raft?
```

---

# 1. Why Do We Need Distributed Systems?

Originally, applications ran on a single machine:

```

```

    Client

    |

  Application

    |

   Database

```

```

This model is simple.

However, as systems grow, one machine becomes insufficient.

There are three main reasons why we need distributed systems.

---

## 1.1 Scalability

A single machine has limited:

- CPU
- Memory
- Storage
- Network bandwidth

When traffic increases, we need more machines:

```

```

    Load Balancer

    /            \

    Server A       Server B

```

```

This allows horizontal scaling.

---

## 1.2 Availability

A single server creates a single point of failure.

If:

```

```

   Server

    X

```

```

the entire system becomes unavailable.

With multiple nodes:

```

```

    Load Balancer

    /          \

  Node A        Node B

```

```

one node can fail while others continue serving requests.

---

## 1.3 Fault Tolerance

In distributed systems:

failure is normal.

Machines can:

- crash
- lose network connectivity
- restart
- have hardware failures

Therefore:

> A distributed system must be designed assuming failures will happen.

---

# 2. The First Problem: Data Consistency

Replication improves availability.

However, replication introduces a new problem.

Suppose we have two replicas:

```

```

    Client

    |

    +---------------+

    Node A       Node B

    x = 10       x = 10

```

```

A client writes:

```

x = 20
```

Node A updates first:

```

Node A:

x = 20

Node B:

x = 10
```

Now the system has inconsistent state.

The question becomes:

> Which value is the correct one?

This is the consistency problem.

---

# 3. What Is CAP Theorem?

CAP describes three properties:

```

C - Consistency

A - Availability

P - Partition Tolerance
```

---

# 3.1 Consistency

Every read receives the latest value.

Example:

After:

```

write x = 20
```

all nodes should return:

```

x = 20
```

---

# 3.2 Availability

Every request receives a response.

The system continues operating even when some nodes fail.

---

# 3.3 Partition Tolerance

The system continues working even when network communication between nodes is broken.

Example:

```

Node A
```

 X

```

Node B
```

Both nodes are alive, but cannot communicate.

---

# 4. Why Can't We Have C + A + P?

This is the core idea behind CAP.

Imagine:

```

Node A -------- Node B
```

  Network Partition

```

```

A client sends:

```

write x = 100
```

Node A receives it:

```

Node A:

x = 100
```

But Node B cannot receive the update:

```

Node B:

x = 50
```

Now another client reads from Node B.

What should it return?

```

100?
```

or

```

50?
```

---

## Option 1: Preserve Consistency (CP)

The system waits until nodes synchronize:

```

Node A

|
|
sync

|

Node B
```

The data is consistent.

However, during the partition:

the system may reject requests.

We sacrifice Availability.

This is CP:

```

Consistency
+
Partition Tolerance
```

---

## Option 2: Preserve Availability (AP)

The system continues serving requests.

However, replicas may temporarily disagree.

This is AP:

```

Availability
+
Partition Tolerance
```

---

# 5. How Do Real Systems Choose?

Different systems have different priorities.

---

# CP Systems

## etcd

Kubernetes stores critical cluster state in etcd:

```

Pod
Node
Deployment
Secret
Configuration
```

Imagine two Kubernetes controllers making conflicting decisions because they see different cluster states.

The result could be:

- duplicate scheduling
- incorrect configuration
- broken cluster state

Therefore etcd prioritizes consistency.

etcd chooses:

```

CP
```

---

## ZooKeeper

ZooKeeper is commonly used for:

- metadata management
- leader election
- distributed coordination

These operations require strong consistency.

Therefore ZooKeeper also chooses:

```

CP
```

---

# AP Systems

## Redis

Redis replication is usually asynchronous:

```

Primary

|
|
Replica
```

The replica may temporarily lag:

```

Primary:

x = 10

Replica:

x = 9
```

Redis prefers availability and performance.

---

## Kafka

Kafka also allows temporary inconsistency between:

- leader
- followers

Because Kafka focuses on:

- throughput
- scalability
- event streaming

It accepts eventual consistency.

---

# 6. If We Choose CP, How Do We Achieve Consistency?

Now we have another question:

> If multiple nodes must agree, who decides the final value?

For example:

```

Node A
Node B
Node C
```

A client writes:

```

set x = 10
```

How do all nodes agree?

This is the consensus problem.

---

# 7. What Is Consensus?

Consensus means:

> Multiple distributed nodes agree on one value even when some nodes fail.

Example:

```

Node A: vote x = 10

Node B: vote x = 10

Node C: vote x = 10
```

The final decision:

```

x = 10
```

---

# 8. Paxos and Raft

Historically, Paxos was one of the earliest consensus algorithms.

It is theoretically powerful.

However, implementing Paxos correctly is difficult.

Raft was designed with a different goal:

> Make consensus understandable.

---

# 9. How Does Raft Work?

Raft divides consensus into three major problems.

---

## 9.1 Leader Election

Nodes have three states:

```

Follower

Candidate

Leader
```

Election process:

```

Follower

|
timeout

v

Candidate

|
majority vote

v

Leader
```

A leader coordinates all writes.

---

## 9.2 Log Replication

Client sends:

```

set x = 10
```

Leader writes:

```

Log Entry:

x = 10
```

Then replicates:

```

```

    Leader

    /    |    \

Node B Node C Node D

```

```

When a majority acknowledges:

the entry becomes committed.

---

## 9.3 Safety

Raft guarantees that committed data cannot diverge.

The system should never reach:

```

Node A:

x = 10

Node B:

x = 20
```

for the same committed state.

---

# 10. How Does etcd Use Raft?

Kubernetes architecture:

```

kubectl

|

API Server

|

etcd

|

Raft Consensus

|

Multiple etcd Nodes
```

When Kubernetes updates state:

```

Write Request
```

  |

Leader

  |

```

Raft Log
```

  |

```

Majority Replication
```

  |

Commit

  |

```

Update Cluster State
```

---

# 11. The Complete Reasoning Chain

The evolution of distributed systems:

```

Need more machines
```

    |

    v

```

Need replication
```

    |

    v

```

Replication creates consistency problems

```

    |

    v

```

CAP theorem

```

    |

    v

```

Choose CP or AP

```

    |

    v

```

CP requires consensus

```

    |

    v

```

Raft / Paxos

```

    |

    v

```

Systems like etcd and ZooKeeper

```

---

# Conclusion

Distributed systems were not created because engineers wanted complicated algorithms.

They were created because:

- one machine is not enough
- failures are unavoidable
- data must be replicated

CAP explains the fundamental tradeoff.

Consensus algorithms like Raft provide a way to build strongly consistent distributed systems.

Understanding this chain explains why:

- Kubernetes needs etcd
- etcd needs Raft
- some systems prefer availability over consistency

The technologies are not isolated concepts.

They are solutions to problems that appeared one after another.
