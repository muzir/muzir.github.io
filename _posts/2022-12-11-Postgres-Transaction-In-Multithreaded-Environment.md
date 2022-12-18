---
layout: post
title: Postgres Transactions In Multithreaded Environment  
background: '/img/posts/background_postgres_transactions.jpg'
fullview: false
---

Few days ago, I was reviewing a simple change in one of our team's repository. It was a one line change in one of our main domain object. Simply inside a transaction there are
below operations;

```
start transaction

Order order = getOrderFromDatabase(orderId);
order.setState(IN_PROGRESS)
update(order)
/*
* some other operations
*/ 
Order order = getOrderFromDatabase(orderId);
sendOrder(order)

end transaction
```

The code block is critical for our business use case. Also I had confused especially about the second `getOrderFromDatabase` method.
The question was how the database behaves in that case where data updated however transaction was not committed yet. Before proceed with the deeper investigation
better to give some context our tech stack. This is a Spring boot java application and we use postgres as a database. Application has the [Spring Transaction Manager](https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/html/transaction.html#transaction-strategies) which covers insert and update
statements of the order domain object.

<script src="https://gist.github.com/muzir/c9ed515268067fb80440553b6056e332.js"></script>

### Deeper Investigation

First I had checked the spring transaction manager default isolation level, according to [Spring documentation default transactional settings](https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/html/transaction.html#transaction-declarative-txadvice-settings)
, it is `DEFAULT`. However it is not clear what is the meaning of `DEFAULT`. Because there is no `DEFAULT` isolation level in [RDBMS isolation levels](https://en.wikipedia.org/wiki/Isolation_(database_systems)#Isolation_levels).
So I need to check [Spring java doc in here](https://docs.spring.io/spring-framework/docs/5.0.x/javadoc-api/org/springframework/transaction/annotation/Isolation.html#DEFAULT), according to 
its java doc it uses the default isolation level of the underlying datastore. Then the question is what is the Postgres database default isolation level? According to [Postgres documentation it is READ COMMITTED](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED).
If its default isolation level is READ COMMITTED I shouldn't see any change yet in the second `getOrderFromDatabase`, because transaction has not committed yet. However according to
our logs that doesn't the case. When I log the `order` object after second get from database I see that status has updated to `IN_PROGRESS`, why?  

```
start transaction
// First get from database before status update
Order order = getOrderFromDatabase(orderId);
order.setState(IN_PROGRESS)
update(order)
/*
* some other operations
*/ 
// Second get from database after status update 
Order order = getOrderFromDatabase(orderId);
sendOrder(order)

end transaction
```

### Mystery Resolving

To answer the above `why?` I need to read the postgres READ COMMITTED documentation a bit more carefully, [in here it states as](https://www.postgresql.org/docs/current/transaction-iso.html#:~:text=Read%20Committed%20is%20the%20default,query%20execution%20by%20concurrent%20transactions)

```
Read Committed is the default isolation level in PostgreSQL. 
When a transaction uses this isolation level, 
a SELECT query (without a FOR UPDATE/SHARE clause) sees only 
data committed before the query began; 
it never sees either uncommitted data or changes committed 
during query execution by concurrent transactions. 
In effect, a SELECT query sees a snapshot of the database as 
of the instant the query begins to run. However, SELECT does see 
the effects of previous updates executed within its own transaction, 
even though they are not yet committed. 
```

So the answer of my question is exactly here, there is a different behaviour for concurrent and own transaction. Let's test that behaviour 
in multithreaded environment.

### Testing Postgres Transactions

I need to create a test scenario to check the difference between concurrent and own transactions behaviour. To do that I create below scenarios   

#### Testing with Single Thread

First create the order and save it to the database. Then in a single thread(`thread1`);

1. Wait 100 miliseconds
2. Log the status of the order
3. Set the order status to `IN_PROGRESS` and updated.
4. Get the updated order object and log the status

<script src="https://gist.github.com/muzir/a9157825af4bccae16306a0ae62628f9.js"></script>

Let's run the test `testReadCommittedIsolationLevel_withSingleTransaction`, here is the logs;

```
thread1 is starting
thread1 - orderAfterInsert orderStatus= NEW
thread1 is updated
thread1 - orderAfterUpdate orderStatus= IN_PROGRESS
thread1 is committing
```

So it proves that `SELECT` reads uncommited changes in its own transaction, because orderStatus has changed from `NEW` to `IN_PROGRESS` after update. 

#### Testing with Multi Threads

Now let's test it with multithreaded environment. I use the same `thread1` method with `therad2`. Inside the `thread2`;

1. Log the status of the order.
2. Set the order status to `IN_PROGRESS` and updated.
3. Get the updated order object and log the status
4. Wait 500 miliseconds.

<script src="https://gist.github.com/muzir/c7bc9f19419aa269f9a0b476310051ee.js"></script>

Let's run the test `testReadCommittedIsolationLevel_withMultipleThreads`, here is the logs;

```
2022-12-18 15:49:01.845 : thread2 is starting
2022-12-18 15:49:01.856 : thread2 - orderAfterInsert orderStatus= NEW
2022-12-18 15:49:01.859 : thread2 is updated
2022-12-18 15:49:01.862 : thread2 - orderAfterUpdate orderStatus= PROCESSED
2022-12-18 15:49:01.950 : thread1 is starting
2022-12-18 15:49:01.953 : thread1 - orderAfterInsert orderStatus= NEW
2022-12-18 15:49:02.365 : thread2 is committing
2022-12-18 15:49:02.368 : thread1 is updated
2022-12-18 15:49:02.371 : thread1 - orderAfterUpdate orderStatus= IN_PROGRESS
2022-12-18 15:49:02.372 : thread1 is committing
```

Let's explain the logs step by step;

1. `thread1` and `thread2` nearly start at the same time.
2. `thread1` sleeps 100ms, in this time thread2 has updated orderStatus as `PROCESSED`.
3. `thread1` starts and log the orderStatus because `thread2` has not committed yet `thread1` log it as `NEW`, `READ COMMITTED` applies.
4. `thread2` committed and orderStatus should be committed as `PROCESSED`.
5. `thread1` updated the orderStatus and then log it as `IN_PROGRESS`
   5.1. That step is critical because `thread2` committed and orderStatus should be `PROCESSED` however thread1 updated the record `3ms` later as `IN_PROGRESS` 
6. `thread1` committed.

### Notes

Threads are good way to test multithreaded environment with integration tests. Relying on the official documentation of the frameworks, 
libraries or databases is the best practice on investigations. 


### Result

Postgres transactions have different isolation level behaviour based on its own transactions and concurrent transactions. Its own transaction they behave like 
`READ UNCOMMITTED` even if their default isolation level is `READ COMMITTED`.

You can find the [all project on Github](https://github.com/muzir/softwareLabs/tree/master/spring-boot-containers)

### References

[Spring transaction manager strategies](https://docs.spring.io/spring-framework/docs/4.2.x/spring-framework-reference/html/transaction.html#transaction-strategies)
[Spring Doc DEFAULT](https://docs.spring.io/spring-framework/docs/4.1.5.RELEASE/spring-framework-reference/html/transaction.html#transaction-declarative-attransactional-settings)
[Spring Java doc default isolation level](https://docs.spring.io/spring-framework/docs/5.0.x/javadoc-api/org/springframework/transaction/annotation/Isolation.html#DEFAULT)
[What is the default isolation level of the Postgres database?](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED)
[Background picture reference](https://www.reddit.com/r/Elephants/comments/si1duq/african_elephants_anup_shahscience_photo_library/)

Happy coding :) 


