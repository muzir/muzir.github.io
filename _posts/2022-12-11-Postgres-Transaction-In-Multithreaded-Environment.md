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
order.setState(PROCESSED)
update(order)
/*
* some other operations
*/ 
Order order = getOrderFromDatabase(orderId);
sendOrder(order)

end transaction
```

Even if one has changed, the code block is critical for our business use case. Also I had confused especially about the second `getOrderFromDatabase` method.
The question was how the database behaves in that case where data updated however transaction was not committed yet. Before proceed with the deeper investigation
better to give some context our tech stack. This is a Spring boot java application and we use postgres as a database.  

### Deeper Investigation



### Mystery Resolving


### Testing Postgres Transactions

#### Testing with Single Thread

#### Testing with Multi Threads


### Notes




### Result




### References

[Background picture reference](https://www.reddit.com/r/Elephants/comments/si1duq/african_elephants_anup_shahscience_photo_library/)

Happy coding :) 


