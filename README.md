# Report on problem: Banking Transaction History and Account Statements

## Design the database schema.

### High level System architecture
![alt text](https://github.com/Suruta/highloaded-apps-task-4/blob/main/files/High-level-System-architecture.drawio.svg?raw=true)

### DB schema architecture
![alt text](https://github.com/Suruta/highloaded-apps-task-4/blob/main/files/DB-schema-architecture.drawio.svg?raw=true)

## Would you store account balances separately from transactions?

Yes. Store balance in separate tables
```sql
SELECT available_balance
FROM account_balances
WHERE account_id = 123456;
```
Advantages:
- Fast balance retrieval.
- No need to scan historical transactions.
- Suitable for high-throughput banking applications.
Disadvantages:
- Balance and ledger must be updated atomically.
- Bugs in update logic can create inconsistencies.

Use both:
- `ledger_entries` stores the permanent financial history.
- `account_balances` stores the current balance.

Both must be updated in the same database transaction.
If the transaction fails, both updates must roll back.

## Would you shard transactions by account ID or transaction date?

We have:
- 30 million accounts.
- 500 million transactions every month.
- 6 billion transactions every year.
One database machine will eventually become a storage and performance bottleneck.

Partition by account id
```
shard = hash(account_id) mod N
```

Usually when quering we ask: "give me all transactions for account X", "what is the balance of account X". Transactions and balance will be in one shard, which means fast lookup. No need to visit other shards.

## How would you implement primary-replica replication?

One primary, one sync replica, other two async replicas

## Should statement queries use read replicas?

Yes. Statement generation is a read-heavy workload. Running all statement queries against the primary could reduce the capacity available for financial writes.
Therefore, I would route statement queries to replicas.
However, there is an important problem: replication lag.
We would solve it with monotonic reads. Once client reads new data we keep it as it is and don't show the old data.

## How would you guarantee consistent statements?

Using Strong serialized isolation.

## How would you handle late-arriving transactions

If transaction is arriving late, we would

## How would you archive old transactions?

Keeping all old transactions it's very costy, on top of that if we had indexing, even more costy. Long backup duration.
We would identify 0-12 month as active, 1 year or more as historical db.

## How would you recover from primary database failure?

Db shardener detects fail of primary through heart beat check.
We can choose one of its sync followers replication to be leader, cuz it will be always up to date.

In case if one of the primary dbs fails, We will choose a new db from read replicas based on who has most latest data. Read replicate becomes a new leader.

## Challenge

A customer requests a statement at 12:00.
At 12:00:01, a new transaction is committed on the primary but hasn't reached the replica.

### Answer

When the customer performs any write, the application records the leader's replication log position. The next statement request carries this position and is routed to a replica whose applied log position ≥ that value. If no replica has caught up, the read falls back to the leader.

The statement query runs under snapshot isolation. All tables are read from the same snapshot taken at query start. This guarantees no read skew

```
last_txn.balance_after  ==  account.balance_amount   (in the same snapshot)
```
If they match → statement is internally consistent, ship it
If they don't → the replica is in an inconsistent intermediate state, reject and retry







