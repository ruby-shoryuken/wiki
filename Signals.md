## TTIN

TTIN tells Shoryuken to print backtraces for all threads in the process along with the current number of busy and ready processors and the actual weights of the queues.

```bash
kill -TTIN [shoryuken_pid]
```

## TSTP

TSTP tells Shoryuken to shut down gracefully. It will stop sending any new requests to fetch from SQS, but will ensure that all the requests already sent will be processed (guaranteeing that no message stayed as non-visible on SQS).

This signal does not exit Shoryuken, you still need to TERM for exiting the process.

```bash
kill -TSTP [shoryuken_pid]
```

## USR1

USR1 tells Shoryuken to shut down gracefully. It will stop sending any new requests to fetch from SQS, but will ensure that all the requests already sent will be processed (guaranteeing that no message stayed as non-visible on SQS). 

Shoryuken exits once everything is processed.

```bash
kill -USR1 [shoryuken_pid]
```

## TERM

TERM tells Shoryuken to hard shutdown within a `timeout`. Any processors that do not finish within the timeout are hard terminated and their messages are pushed back to AWS SQS. The timeout defaults to 8 seconds since all Heroku processes must exit within 10 seconds.

The `timeout` can be defined through the `-t` option or in the `-C shoryuken.yml` config by setting the `timeout: N` value.

```bash
kill -TERM [shoryuken_pid]
```