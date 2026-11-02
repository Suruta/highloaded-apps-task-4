# Problem: Banking Transaction History and Account Statements

You are designing a banking transaction history service.

Requirements:
- 15 million customers.
- 30 million accounts.
- 500 million transactions per month.
- Transactions are continuously created.
- Customers frequently request account statements.
- Statements must contain accurate transaction data.
- The system must support monthly and yearly reports.
- The service must remain available during database failures.

Example:
Account: 123456

Transactions:
January 1:  +100,000 KZT
January 3:   -20,000 KZT
January 5:   -15,000 KZT
January 8:   +50,000 KZT
Current balance: 115,000 KZT

Your tasks:
1. Design the database schema.
2. Would you store account balances separately from transactions?
3. Would you shard transactions by account ID or transaction date?
4. How would you implement primary-replica replication?
5. Should statement queries use read replicas?
6. How would you guarantee consistent statements?
7. How would you handle late-arriving transactions
8. How would you archive old transactions?
9. How would you recover from primary database failure?

Challenge:
A customer requests a statement at 12:00.
At 12:00:01, a new transaction is committed on the primary but hasn't reached the replica.

The statement is generated from the replica.
How do you guarantee that the statement is accurate and internally consistent?

## Criteria

1. System architecture and component design
2. Database schema, ERD, and SQL examples
3. Application of database concepts (transactions, isolation, locking, replication, sharding, etc.)
4. Failure scenarios and recovery strategies
5. Presentation, technical explanations, and answers to questions

## Important Requirements

You are encouraged to use diagrams, SQL examples, and practical scenarios to support your explanations.
