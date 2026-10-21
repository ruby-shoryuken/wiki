The `shoryuken sqs` commands manage SQS queues from the command line. To list them, run `shoryuken help sqs`.

| Command | Effect |
|---------|--------|
| `ls [QUEUE-NAME-PREFIX]` | Lists queues |
| `dump QUEUE-NAME` | Writes the messages of a queue to a JSON lines file |
| `requeue QUEUE-NAME PATH` | Sends the messages of a dump file to a queue |
| `mv QUEUE-NAME-SOURCE QUEUE-NAME-TARGET` | Moves messages from one queue to another |
| `purge QUEUE-NAME` | Deletes the messages in a queue |
| `create QUEUE-NAME` | Creates a queue |
| `delete QUEUE-NAME` | Deletes a queue |

## Queue names and the endpoint

The commands take a queue name, not a queue URL. Shoryuken looks up the URL of the name. On SQS, an unknown name fails with `The specified queue <name> does not exist`.

`-e`/`--endpoint` sets the SQS endpoint for every command. Its default is the `SHORYUKEN_SQS_ENDPOINT` environment variable. See [[Using a local mock SQS server]].

For the IAM actions that the commands need, see [[Amazon SQS IAM Policy for Shoryuken]].

## ls

```bash
shoryuken sqs ls [QUEUE-NAME-PREFIX]
```

Prints a table of the queues whose names start with the prefix, or of all queues without a prefix. It shows at most 1,000 queues, the SQS limit for one `ListQueues` call. The columns are `Queue`, `Messages Available`, `Messages Inflight` and `Last Modified`.

| Option | Default | Effect |
|--------|---------|--------|
| `-w`, `--watch` | | Prints the table again until you press Ctrl-C |
| `-n`, `--interval` | `2` | Seconds between two tables with `--watch` |

## dump

```bash
shoryuken sqs dump QUEUE-NAME
```

Receives the messages of the queue and writes them to `<queue-name>-<date>.jsonl`, one message per line. The date is today's date, for example `my-queue-2026-10-19.jsonl`.

| Option | Default | Effect |
|--------|---------|--------|
| `-n`, `--number` | all | The number of messages to dump |
| `-p`, `--path` | `./` | The directory for the dump file |
| `-d`, `--delete` | `true` | Deletes the dumped messages from the queue. Use `--no-delete` to keep them |

`--number` must be at least 1, else `dump` fails with `Number of messages must be at least 1, got <n>`. `--path` must be an existing directory that you can write to and enter, else `dump` fails with `Path <path> is not a directory` or `Path <path> is not writable`. `dump` checks both before it receives any message.

If the dump file already exists, `dump` fails with `File <path> already exists`. A dump that fails or is stopped while it writes removes its file. Only a completed dump blocks the file name. So you can dump a queue only once per day into the same directory. Use another `--path` for a second dump.

Binary message attribute values are written in base64, with `"binary_value_encoding": "base64"` next to the value.

When a delete from the queue fails, `dump` prints `Could not delete <id>, code: <code>, message: <message>, sender_fault: <sender_fault>` for each message. It keeps the dump file, then fails with `Could not delete <n> dumped messages from <queue>` (exit status 1).

An empty queue gives `Queue <name> is empty`, and no file is written.

## requeue

```bash
shoryuken sqs requeue QUEUE-NAME PATH
```

Sends the messages of a dump file to the queue. It does not change the file.

| Option | Default | Effect |
|--------|---------|--------|
| `-n`, `--batch_size` | `10` | The maximum number of messages in one send batch |

Shoryuken also keeps each batch within the SQS batch payload limit (1 MiB).

When the queue rejects a message, `requeue` prints `Could not requeue <id>, code: <code>` for it. The `Requeued <n> messages` line counts only the accepted messages. `requeue` then fails with `Could not requeue <n> messages from <path> to <queue>` (exit status 1).

For a message from a FIFO queue, `requeue` sends a new `MessageDeduplicationId` and keeps its `MessageGroupId`. So requeueing the same dump file twice delivers its messages twice.

`requeue` decodes the base64 Binary values. It also reads dump files without `binary_value_encoding`, whose Binary values it sends as they are.

## mv

```bash
shoryuken sqs mv QUEUE-NAME-SOURCE QUEUE-NAME-TARGET
```

Receives the messages of the source queue and sends them to the target queue.

| Option | Default | Effect |
|--------|---------|--------|
| `-n`, `--number` | all | The number of messages to move |
| `-d`, `--delete` | `true` | Deletes the moved messages from the source queue. Use `--no-delete` to keep them |

`--number` must be at least 1, else `mv` fails with `Number of messages must be at least 1, got <n>` before it receives any message.

When the target queue rejects a message, `mv` prints `Could not requeue <id>, code: <code>` for it and does not delete it from the source queue. The `Moved <n> messages` line counts only the accepted messages. `mv` then prints `Could not move <n> messages, they stay in <source>` and exits with status 1.

When a delete from the source queue fails, `mv` prints `Could not delete <id>, code: <code>, message: <message>, sender_fault: <sender_fault>` for each message, then `Could not delete <n> moved messages from <source>`, and exits with status 1.

For a message from a FIFO queue, `mv` sends a new `MessageDeduplicationId` and keeps its `MessageGroupId`.

## Draining a queue with dump and mv

`dump` and `mv` delete the messages only after they have received all of them. Until then, the received messages are in flight. This has two effects:

- A queue with more messages than the SQS in-flight limit (about 120,000 for a standard queue, 20,000 for a FIFO queue) cannot be drained in one run. Run the command again for the rest. No message is lost.
- On a FIFO queue, SQS returns no more messages of a message group while the group has messages in flight. So one run takes at most one receive (up to 10 messages) per message group. Run the command again until the queue is empty.

A second `dump` of the same queue on the same day needs another `--path` (see [dump](#dump)).

## purge

```bash
shoryuken sqs purge QUEUE-NAME
```

Sends a purge request for the queue. SQS takes up to 60 seconds to delete the messages.

## create

```bash
shoryuken sqs create QUEUE-NAME
shoryuken sqs create my-queue.fifo -a ContentBasedDeduplication:true
```

| Option | Default | Effect |
|--------|---------|--------|
| `-a`, `--attributes` | `{}` | Queue attributes as `Name:value` pairs |

A name that ends with `.fifo` sets `FifoQueue` to `true`. See [[FIFO Queues]].

## delete

```bash
shoryuken sqs delete QUEUE-NAME
```

Deletes the queue.
