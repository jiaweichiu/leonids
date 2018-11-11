---
layout: post
title:  "Consistency models"
categories: [systems]
comments: false
math: true
excerpt: "Understand the various consistency models out there."
---

Let's take a look at this "graph" of [consistency models at jepsen.io](https://jepsen.io/consistency).

![consistency models from jepsen.io](/img/jepsen_consistency.png)

The arrow means "is implied by". For example, "strict serializable" implies both "serializable" and "linearizable".

As for the coloring:
* Pink means "unavailable" during some types of network failures;
* Orange means "sticky available", i.e., non-faulty nodes continue to be available, assuming that clients stick to the same servers they are already talking to;
* Blue means "total available", which means every non-fault node is available, even when the network is completely down.

# Linearizable versus serializable

Peter Bailis gave a [nice summary](http://www.bailis.org/blog/linearizability-versus-serializability/).

Linearizable: single-operation, single-object, real-time order.
{: .notice}

Linearizability is the C in the CAP theorem. It means writes appear to be instantaneous, i.e., atomic consistency. However, it does not care about transactions.

Serializable: multi-operation, multi-object, arbitrary total order.
{: .notice}

Serializability is about transactions. It is the I (Isolation) in ACID. It guarantees that a set of transactions (over multiple items) is equivalent to some serial execution (total ordering) of the transactions. You cannot interleave sub-operations between transactions. However, serializability does not care about the actual order of transactions. (Note: Order of operations *within* each transaction should be preserved.)

Neither linearizability nor serializability is achievable without expensive coordination. This implies no full availability. And in practice, systems implement weaker versions of these for performance purposes.

Finally we have *strict serializability*.

Strict serializable: multi-operation, multi-object, real-time order.
{: .notice}

As we expect, it means transactions appear to execute sequentially (serializable) and in addition, the order is not arbitrary, but in the obvious order --- whichever transaction arrives first gets completed first.

# Linearizable implies sequential consistency

Both sequential consistency and linearizability are about single operations. Sequential consistency is somewhat similar to serializability: it is for single operations instead of transactions. It says that no matter how you execute the operations concurrently, it is as if there is some order or permutation of these operations across all processors.

Formal definition:
> the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program

Sequential consistency only claims there exists such an order of operations (program order). Linearizability is a stronger condition and implies sequential consistency. It says there exists such an order and moreover, this order is the order in which these operations or requests arrive.

# Serializability implies snapshot isolation

Snapshot isolation is a weaker guarantee on transactions than serializability. It is adopted in many databases, e.g., MySQL, Postgres, MongoDB, because it offers much better performance than serializability. It is commonly implemented by MVCC (multiversion concurrency control), i.e., keeping multiple versions of the same value. Common point of confusion: In SQL, "serializable" often means "snapshot isolation" only.

Snapshot isolation means that every transaction operates on an *independent* snapshot of the database. This allows multiple transactions to happen concurrently, based on the snapshots at the start of each transaction. However, if there are conflicts, say transaction 1 needed to modify an object X, but transaction 2 committed a write to X between transaction 1 start and transaction 1 commit,then transaction 1 must abort.

One key feature of snapshot isolation is that when the transaction is done, its changes become visible atomically. (This seems like a basic requirement on transactions!) Hence, snapshot isolation also implies "read committed" which means there are no *dirty reads*, i.e., we cannot observe writes from transactions which have not committed.

Unlike serializability which requires a total order, snapshot isolation only requires a partial order --- sub-operations in transactions may interleave in arbitrary ways as long as there are no conflicts between the transactions.

Remark: If I were to build a database, I will probably go with this model as well.
