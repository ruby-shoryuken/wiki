By default Shoryuken does not support hot-reload for rails.
If you want to enable hot-reload for the dev environment, you need to configure it as follows.
```ruby
# config/initializers/shoryuken.rb
Shoryuken.enable_reloading = true
```
This assumes that you have [Rails reloading](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html#reloading) enabled (which is enabled in dev by default), and that you start Shoryuken with `-R`. Shoryuken sets its Rails reloader hook only when it loads Rails itself.