By default Shoryuken does not support hot-reload for rails.
If you want to enable hot-reload for the dev environment, you need to configure it as follows.
```Ruby
# config/initializers/shoryuken.rb
Shoryuken.enable_reloading = true
```
This assumes that you have [rails reloading](https://guides.rubyonrails.org/v7.1/autoloading_and_reloading_constants.html#reloading) enabled (which is enabled in dev by default).