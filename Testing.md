There are few options for testing Shoryuken workers, there are some examples below with RSpec, which probably works very similar with most of the testing frameworks.

Another alternative is to enable the [Shoryuken Inline adapter](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-Inline-adapter), or if you are using Active Job, refer to [Active Job Inline adapter
](http://api.rubyonrails.org/classes/ActiveJob/QueueAdapters/InlineAdapter.html).

## Testing workers with RSpec

For testing with RSpec you can stub `perform_async` and `perform_in`:

```ruby
# expecting the worker to be called called
expect(MyJob).to receive(:perform_async)

# calling the original worker
allow(MyJob).to receive(:perform_async) { |sqs_msg, body| MyJob.new.perform(sqs_msg, body) }
```

### Full testing example

```ruby
require 'spec_helper'
# OR
# require 'rails_helper' # with Rails

RSpec.describe MyWorker do
  let(:sqs_msg) { double message_id: 'fc754df7-9cc2-4c41-96ca-5996a44b771e',
                  body: 'test',
                  delete: nil }

  let(:body) { 'test' }

  describe '#perform' do
    subject { MyWorker.new }

    it 'prints the body message' do
      expect { subject.perform(sqs_msg, body) }.to output('test').to_stdout
    end

    it 'deletes the message' do
      expect(sqs_msg).to receive(:delete)

      subject.perform(sqs_msg, body)
    end

    it 'pushes a new message' do
      sqs_queue = double 'other queue'

      allow(Shoryuken::Client).to receive(:queues).
        with('other_queue').and_return(sqs_queue)

      expect(sqs_queue).to receive(:send_message).
        with('new test')

      subject.perform(sqs_msg, body)
    end
  end
end
```

## Running workers inline in one test

To run `perform_async` and `perform_in` inline only in some tests, switch the `worker_executor` and restore it afterwards (see [[Shoryuken Inline adapter]]):

```ruby
RSpec.describe 'processing a message' do
  around do |example|
    Shoryuken.worker_executor = Shoryuken::Worker::InlineExecutor
    example.run
  ensure
    Shoryuken.worker_executor = Shoryuken::Worker::DefaultExecutor
  end

  it 'runs the worker right away' do
    MyWorker.perform_async('test')
  end
end
```

## Testing against a local SQS server

For end-to-end tests with real queues, run a local SQS-compatible server. See [[Using a local mock SQS server]].
