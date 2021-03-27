---
layout: post
title:  "Understanding Redis: 4"
author: "Shori"
comments: false
tags: Redis
---

# Implementation of Functionalities: Transactions

Transactions offer a way to package multiple commands into one, and then sequentially execute these commands. When a transaction is executing, it would not yield its execution voluntarily - the server will only start taking up other commands after the transaction has been executed. 

For instance:

``` bash
redis> MULTI
OK
redis> SET book-name "Mastering C++ in 21 days"
QUEUED
redis> GET book-name
QUEUED
redis> SADD tag "C++" "Programming" "Mastering Series"
QUEUED
redis> SMEMBERS tag
QUEUED
redis> EXEC
1) OK
2) "Mastering C++ in 21 days"
3) (integer) 3
4) 1) "Mastering Series"
   2) "C++"
   3) "Programming"
```
Therefore, a transaction will go through 3 phases from the beginning to execution:

1. Starting the transaction;
2. The commands get queued;
3. Transaction execution.

## 4.1 Transaction start

The command ```MULTI``` flags the starting of a transaction. All it does is toggled the ```REDIS_MULTI``` option in Redis, and the client would switch to transaction mode.

## 4.2 Queueing the commands

By the way, when the client is not in a "transaction mode", all the commands will be sent to the server and be executed instantly, like the examples given before.

On the other hand, when the client is in the "transaction mode", it halts from executing the commands. Instead, it stores them into a "transaction queue" and returns ```QUEUED```.

The transaction queue is intrinsically an array. Each elements holds an command, the arguments of the command, and the count of arguments.

Thus, when we created the transaction queue above in the instance, we have an array like:

| Index | cmd | argv | argc  |
|:---:|:---:|:----:|:---:|
| 0  | ```SET```  | ["book-name", "Mastering C++ in 21 days"]  | 2  |
| 1  | ```GET```  | ["book-name"]  | 1  |
| 2  | ```SADD```  | ["tag", "C++", "Programming", "Mastering Series"]  | 4  |
|3|```SMEMBERS```|["tag"]|4|

## 4.3 Transaction execution

All the commands would be pushed to the transaction queue as shown above, except the four transaction commands, ```EXEC```, ```DISCARD```, ```MULTI```, and ```WATCH```. Those four commands will be executed instantly by the server.

When the transaction is executed, the server will create an reply array, which includes all the return values of each command, as the final return value.

## 4.4 ```DISCARD```, ```MULTI```, and ```WATCH```

```DISCARD``` is used to cancel a transaction. It will clear the transaction queue and restore the client to non-transaction mode.

And, you cannot invoke a ```MULTI``` within a ```MULTI```.

### ```WATCH```

```WATCH``` is used before the client enters transaction mode to a specific key.

When ```WATCH``` is triggered, it will start monitoring whether any key being monitored is changed. If any change is noticed, it will cancel the entire transaction.

Redis maintains a ```watched_keys``` dict for ```WATCH```. The dict keeps the keys that are being watched, and the value of the a key is a linked-list of all the clients that are currently watching the key.

![](../assets/redis-4/watch1.png)

When a command that changes the key space of the database is successfully executed, the ```multi.c/touchWatchKey``` function will be invoked to see whether any key in the ```watched_keys``` dict is changed. If so, the function will trigger the ```REDIS_DIRTY_CAS``` flag in all the clients watching the key.

Then, when a client sends ```EXEC``` to execute  command, the server will check the ```REDIS_DIRTY_CAS``` flag of the client. If it's triggered, the server will discard the transaction and returns null.

## 4.5 ACID properties

In traditional relational databases, ACID properties, (Atomicity, Consistency, Isolation, Durability) are often used to verify the security of the database. While in Redis transaction, the Consistency and Isolation are preserved, while the other two are not ensured.

### 4.5.1 Atomicity

A single command in Redis is atomic. However, Redis doesn't ensure the atomicity of transactions.

If a transaction gets interrupted, the successfully finished commands does not roll back. There is [an article](https://redislabs.com/blog/you-dont-need-transaction-rollbacks-in-redis/) from RedisLabs defending no rollback offered.

### 4.5.2 Consistency

 If there's a **queueing error**, for example, a mismatch in argument number in the command, you get an error response from the server instantly. And the command won't be executed when ```EXEC``` is invoked.

``` bash
redis 127.0.0.1:6379> MULTI
OK
redis 127.0.0.1:6379> set key
(error) ERR wrong number of arguments for 'set' command
redis 127.0.0.1:6379> EXISTS key
QUEUED
redis 127.0.0.1:6379> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
```

If there's an **execution error**, for example, a mismatch in type and command, you get an error regarding the command in the return message at the end of the transaction execution. The error does not affect other commands.

#### Persistence options

If the Redis service process gets terminated, according to the persistence option the administrator selected, you get different behavior.

When **no persistence** option is selected, everything is stored in memory. When the process gets terminated, consistency is preserved by losing everything (how grim).

The **RDB persistence** performs point-in-time snapshots of your dataset at specified intervals.

The **AOF persistence** logs every write operation received by the server, that will be played again at server startup, reconstructing the original dataset. Commands are logged using the same format as the Redis protocol itself, in an append-only fashion. Redis is able to rewrite the log in the background when it gets too big.

It is possible to combine both AOF and RDB in the same instance. Notice that, in this case, when Redis restarts the AOF file will be used to reconstruct the original dataset since it is guaranteed to be the most complete.

### 4.5.3 Isolation

*Needs to be checked with the latest version.*

### 4.5.4 Durability

Redis doesn't ensure durability with any of its persistence options. The closest being when using AOF perisistence. But as AOF logs are being stored using a background process, the Redis process does not block when the logs are being saved, which allows a small period of time that the AOF is unsynced.