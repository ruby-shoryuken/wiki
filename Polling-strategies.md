Shoryuken has built in two polling strategies `WeightedRoundRobin` and `StrictPriority`, but you are free to extend it and use your own polling strategy in case you have a special need.

## WeightedRoundRobin

`WeightedRoundRobin` is the default polling strategy for all groups, it works as follows: 

Given the configuration below:

```yaml
polling_strategy: WeightedRoundRobin # Not necessary, WRR is the default
queues:
  - [queue1, 8]
  - [queue2, 4]
  - [queue3, 1]
```

A queue weight starts at 1, and after every fetch returning messages, it gets increased one-by-one until it reaches the max weight. In this example, the max weight for `queue1` is 8.

Supposing all queues are full of messages, how it would look like:

1. fetches messages from `queue1`
2. fetches messages from `queue2`
3. fetches messages from `queue3`
4. fetches messages from `queue1`
5. fetches messages from `queue1`
6. fetches messages from `queue2`
7. fetches messages from `queue2`
8. fetches messages from `queue3`

If `delay` is configured:

```yaml
delay: 10
queues:
  - [queue1, 8]
  - [queue2, 4]
  - [queue3, 1]
```

When a fetch returns no messages, Shoryuken pauses the queue for `delay` seconds (default `0`).

## StrictPriority

`StrictPriority` will cause Shoryuken to keep fetching messages from the highest priority queue until it returns no messages, it will then try the next highest priority queue, etc.

```yaml
polling_strategy: StrictPriority
queues:
  - [queue1, 3]
  - [queue2, 2]
  - [queue3, 1]
```

The queue weight sets the priority: the queue with the highest weight comes first.

While `queue1` has messages, Shoryuken won't try to fetch message from `queue2` or `queue3`.

`delay` is also supported for `StrictPriority`. With `delay` set to 0, Shoryuken does not pause empty queues: every iteration causes Shoryuken to try the queues in priority order, stopping once it finds a job.