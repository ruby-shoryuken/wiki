Add below to some Rails initializer, example: config/initializers/shoryuken.rb
```ruby
# Shoryuken middleware to capture worker errors and send them to Sentry.io
module Shoryuken
  module Middleware
    module Server
      class SentryReporter
        def call(worker_instance, queue, sqs_msg, body)
          Sentry.with_scope do |scope|
            scope.set_tags(job: body['job_class'], queue:)
            scope.set_context(:message, body)
            
            Sentry.with_exception_captured do
              yield
            end
          end
        end
      end
    end
  end
end

Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add Shoryuken::Middleware::Server::SentryReporter
  end
end
```