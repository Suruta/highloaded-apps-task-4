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
A statement query that computes SUM(amount) over millions of rows per account is O(n) and would make every login slow.
Storing the balance gives O(1) balance reads.

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

One primary, one sync replica, other two async replicas.  
One sync replica is needed if primary db crashes.

## Should statement queries use read replicas?

Yes, but with conditions.  
When we want to see our balance we should request it from primary db, also this operation is very cheap, not a big deal for primary db.  
If we want to see our history of payments we should use followers to offload the leader, since having replication lags is not a problem.


## How would you guarantee consistent statements?

Statement query reads a single consistent snapshot taken at the start of the query.  
Even if concurrent writes arrive mid-query, the statement sees a point-in-time view.  
Once a customer reads from a follower replica, keep sending them to that follower, so they never see old data.

## How would you handle late-arriving transactions

`txn_timestamp` = business time(when the event actually happened)  
`posted_at` = system time(when it was recorded)  
Late transactions with `txn_timestamp` before the cutoff are inserted with their original business date  
`balance_after` is computed by recomputing the balance chain from the last confirmed point

## How would you archive old transactions?

Time-based partitioning on the transaction table
After 12–24 months, partitions are detached from the live shard and exported to cold storage.

## How would you recover from primary database failure?

Db shardener detects fail of primary through heart beat check.
We can choose its sync followers replication to be leader, cuz it has always been up to date.

## Challenge

A customer requests a statement at 12:00.  
At 12:00:01, a new transaction is committed on the primary but hasn't reached the replica.

### Answer

When the customer performs any write, the application records the leader's replication log position.  
The next statement request carries this position and is routed to a replica whose applied log position ≥ that value.  
If no replica has caught up, the read falls back to the leader.

The statement query runs under snapshot isolation. All tables are read from the same snapshot taken at query start. This guarantees no read skew

```
last_txn.balance_after  ==  account.balance_amount   (in the same snapshot)
```
If they match → statement is internally consistent, ship it  
If they don't → the replica is in an inconsistent intermediate state, reject and retry







