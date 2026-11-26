An SNS topic can send each published message to several SQS queues (fan-out). Shoryuken processes these queues like any other queue. What your worker gets depends on the `RawMessageDelivery` attribute of the queue's subscription to the topic.

## Turn on raw message delivery

With `RawMessageDelivery` set to `true`, SNS sends the published message as is: the SQS body is the published message, and the SNS message attributes arrive as SQS message attributes. The worker reads the message like one sent straight to SQS:

```ruby
class OrderEmailsWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'order-emails', auto_delete: true, body_parser: :json

  def perform(sqs_msg, order)
    event_type = sqs_msg.message_attributes['event_type']&.string_value
    # order is the published JSON, parsed: {"id" => "A1", "total" => 42}
  end
end
```

`RawMessageDelivery` is `false` by default. Set it on the subscription, for example with `aws-sdk-sns`:

```ruby
sns.set_subscription_attributes(
  subscription_arn: subscription_arn,
  attribute_name: 'RawMessageDelivery',
  attribute_value: 'true'
)
```

With raw message delivery, SNS sends at most 10 message attributes to the queue, and discards a message that has more (see [Amazon SNS raw message delivery](https://docs.aws.amazon.com/sns/latest/dg/sns-large-payload-raw-message-delivery.html)).

## Without raw message delivery

By default, the SQS body is the SNS notification, a JSON document:

- The published message is a String in its `Message` field.
- The SNS message attributes are in its `MessageAttributes` field, each as a `{"Type" => ..., "Value" => ...}` Hash. `sqs_msg.message_attributes` is empty.

With `body_parser: :json`, `perform` gets this whole notification. For a queue that cannot use raw message delivery, use a Proc as the [[body_parser|Worker options#body_parser]]. This one gives `perform` the published JSON, the attribute values and the topic ARN:

```ruby
SNS_ENVELOPE = lambda do |sqs_msg|
  env = JSON.parse(sqs_msg.body)
  attrs = (env['MessageAttributes'] || {}).transform_values { |a| a['Value'] }
  { 'message' => JSON.parse(env['Message']), 'attributes' => attrs, 'topic' => env['TopicArn'] }
end

class OrderAnalyticsWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'order-analytics', auto_delete: true, body_parser: SNS_ENVELOPE

  def perform(sqs_msg, body)
    event_type = body['attributes']['event_type']
    order = body['message']
  end
end
```

This parser raises when the body or its `Message` is not JSON. For what happens to the message then, see [[body_parser|Worker options#body_parser]].

## Several event types on one queue

Without the `shoryuken_class` message attribute, Shoryuken uses the worker registered for the queue (see [[Worker registry]]). When one queue gets several event types, write one worker that branches on the event type attribute:

```ruby
class OrderEventsWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'order-events', auto_delete: true, body_parser: :json

  def perform(sqs_msg, order)
    case sqs_msg.message_attributes['event_type']&.string_value
    when 'order.created' then order_created(order)
    when 'order.cancelled' then order_cancelled(order)
    else Shoryuken.logger.warn("Unknown event type, message #{sqs_msg.message_id}")
    end
  end

  # order_created and order_cancelled are your handlers
end
```

This worker uses raw message delivery. With the `SNS_ENVELOPE` parser, branch on `body['attributes']['event_type']`. A message with an unknown event type is logged and, with `auto_delete: true`, deleted.

## Queue policy on AWS

SNS needs permission to send to the queue. Set a queue policy that allows `sqs:SendMessage` for the `sns.amazonaws.com` service principal, with `aws:SourceArn` set to the topic ARN:

```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "sns.amazonaws.com" },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:REGION:ACCOUNT:order-emails",
      "Condition": {
        "ArnEquals": { "aws:SourceArn": "arn:aws:sns:REGION:ACCOUNT:orders" }
      }
    }
  ]
}
```

See [Subscribing an Amazon SQS queue to an Amazon SNS topic](https://docs.aws.amazon.com/sns/latest/dg/subscribe-sqs-queue-to-sns-topic.html). For the permissions Shoryuken itself needs, see [[Amazon SQS IAM Policy for Shoryuken]].
