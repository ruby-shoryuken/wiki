Shoryuken traps the `INT`, `TERM`, `USR1`, `TSTP` and `TTIN` signals.

| Signal | Effect | Process exits |
|--------|--------|---------------|
| `TTIN` | Logs the backtraces of all threads | No |
| `TSTP` | Graceful stop: stops fetching and waits for in-flight messages | No |
| `USR1` | Graceful stop, then exit | Yes |
| `TERM`, `INT` | Hard shutdown within `timeout` | Yes |

When Shoryuken stops, a message that it has received but not started is made visible again at once, so another consumer can receive it.

## TTIN

TTIN tells Shoryuken to log the backtraces of all threads in the process. Use it to find a hung or slow worker.

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

The wait for busy workers in a graceful stop (USR1 and TSTP) is also limited by `timeout`: workers that do not finish within it are killed, so a hung worker cannot block the shutdown.

## TERM

TERM tells Shoryuken to hard shutdown within a `timeout`. Any processors that do not finish within the timeout are hard terminated and their messages are pushed back to AWS SQS. The timeout defaults to 8 seconds since all Heroku processes must exit within 10 seconds.

The `timeout` can be defined through the `-t` option or in the `-C shoryuken.yml` config by setting the `timeout: N` value.

```bash
kill -TERM [shoryuken_pid]
```

INT (for example Ctrl+C in a terminal) does the same as TERM.

## Lifecycle events

| Signal | Events fired |
|--------|--------------|
| `TSTP`, `USR1` | `quiet`, `shutdown`, `stopped` |
| `TERM`, `INT` | `shutdown`, `stopped` |

A signal that arrives after a stop has completed fires no events. For example, `TERM` after a completed `TSTP` stop only exits the process: the `shutdown` and `stopped` handlers do not run again.

See [[Lifecycle Events]].
