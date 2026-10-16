Below is a minimum IAM Policy required for Shoryuken to function.

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "ShoryukenAccess",
			"Effect": "Allow",
			"Action": [
				"sqs:ChangeMessageVisibility",
				"sqs:DeleteMessage",
				"sqs:GetQueueAttributes",
				"sqs:GetQueueUrl",
				"sqs:ReceiveMessage",
				"sqs:SendMessage",
 				"sqs:ListQueues"
			],
			"Resource": "arn:aws:sqs:REGION:ACCOUNT:*"
		}
	]
}
```

*Note:* `sqs:ListQueues` is only needed for running `shoryuken sqs ls` (for listing queues, see `shoryuken help sqs`). You don't need it for sending and receiving messages.

Some other `shoryuken sqs` commands need more actions: `shoryuken sqs create` needs `sqs:CreateQueue`, `shoryuken sqs delete` needs `sqs:DeleteQueue` and `shoryuken sqs purge` needs `sqs:PurgeQueue`.

See [IAM Policies](http://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html) for the AWS User Guide.