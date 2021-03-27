---
layout: post
title:  "Understanding Redis: 5"
author: "Shori"
comments: false
tags: Redis
---

# Implementation of Functionalities: Subsription and Publishing / Slow Logs

## 5.1 Subscribing and publishing to a channel

The Redis ```SUBSCRIBE``` command can have the client subscribe to any number of channels. When a new message is published to a channel, the message will be sent to all teh clients that have subscribed to the channel.

Seems good. And let's talk about the implementation of ```SUBSCRIBE``` and ```PUBLISH``` commands.

<br>

## 5.2 Implementation

Similar to the ```watched_keys``` mechanism we mentioned in the last chapter, subscription is also implemented with a dict. Where the keys of the dict are channels, while each key links to a value that is a linked-list that contains all the clients that has subsribed to the channel.

Therefore, the ```SUBSCRIBE``` command is roughly implemented as
```python
# pseudocode
def SUBSCRIBE(client, channels):
    redisServer.pubsub_channels[channel].append(client)
```
After learning the dict structure of ```pubsub_channel```, it's straightforward to think of the implementation for ```PUBLISH```, which just traverses the linked-list of clients for the channel, and sent messages accordingly.

And ```UNSUBSCRIBE```, similarly, deletes the client key from the linked-list.

<br>

## 5.3 Subscribing to patterns

It is not mysterious as it sounds. It's just subscribing to channels, while this time, it has pattern matching.

For example, if there are two channels named ```tweet.ideas``` and ```tweet.execution```, then subscribing to a pattern ```tweet.*``` would make the client subscribing to both channels.

Redis keeps a linked-list ```pubsub_patterns``` of clients and the pattern they subscribed to. 

When a message is published to a channel, it would traverse ```pubsub_patterns```. If any pattern matches the channel it is publishing to, it would also notify the client.

<br>

## 5.4 Slow logs

Slow logs are a way of monitoring system performance. It will record the running time of commands, and generate slow logs for those commands.

``` c
typedef struct slowlogEntry {
    robj **argv;
    int argc;
    long long id;
    /* Unique entry identifier. */
    long long duration; /* Time spent by the query, in nanoseconds. */
    time_t time; /* Unix time at which the query was executed. */
} slowlogEntry;
```

Slow log entries are stored in a linked-list within the ```redisServer``` struct. Another field ```slowlog_log_slower_than``` sets the time limit of a command. If the execution of a command exceeds that, it will be added to the slow log linked-list.