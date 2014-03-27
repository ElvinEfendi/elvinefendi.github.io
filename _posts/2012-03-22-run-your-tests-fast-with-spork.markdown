---
layout: post
title: "run your tests fast with spork"
date: 2012-03-22 12:40
comments: true
tags: spork rspec rails
---
Running tests sometimes may take long time. In the post I'll explain how to reduce the time spent to run tests.
For this purpose there is a nice tool called spork. Spork forks a copy of the server each time you run your tests.
To use spork in your Rails application first you should add it to your Gemfile: 

{% highlight ruby %}
...

group :test do
  ...
  gem 'spork', '~> 1.0rc'
  ...
end

...
{% endhighlight %}

Or certainly you can use gem install command as well.
Then run bundle install. After installing the gem you should configure it. In this post I will show configuration with rspec,
therefor you should have rspec among your gems. To configure spork with rspec run the following command(you should be in your application main directory).

{% highlight ruby %}
spork rspec --bootstrap
{% endhighlight %}

Now your app's spec_helper.rb file should be as following:
{% highlight ruby %}
require 'rubygems'
require 'spork'
#uncomment the following line to use spork with the debugger
#require 'spork/ext/ruby-debug'

Spork.prefork do
  # Loading more in this block will cause your tests to run faster. However,
  # if you change any configuration or code from libraries loaded here, you'll
  # need to restart spork for it take effect.


  # This file is copied to spec/ when you run 'rails generate rspec:install'
  ENV["RAILS_ENV"] ||= 'test'
  require File.expand_path("../../config/environment", __FILE__)
  require 'rspec/rails'

  # Requires supporting ruby files with custom matchers and macros, etc,
  # in spec/support/ and its subdirectories.
  Dir[Rails.root.join("spec/support/**/*.rb")].each {|f| require f}
end
Spork.each_run do
  # This code will be run each time you run your specs.

end

# instructions will be here

RSpec.configure do |config|
# == Mock Framework
#
# If you prefer to use mocha, flexmock or RR, uncomment the appropriate line:
#
# config.mock_with :mocha
# config.mock_with :flexmock
# config.mock_with :rr
config.mock_with :rspec

config.before(:suite) do
DatabaseCleaner.strategy = :truncation
end

config.before(:each) do
DatabaseCleaner.start
#load "#{Rails.root}/db/seeds.rb"
end

config.after(:each) do
DatabaseCleaner.clean
end

end

{% endhighlight %}

Then read all the instructions. As noted in instructions we should move all appropriate loading to spork each block. So our new spec_helper.rb file
should be as below:
{% highlight ruby %}
require 'rubygems'
require 'spork'
#uncomment the following line to use spork with the debugger
#require 'spork/ext/ruby-debug'

Spork.prefork do
  # Loading more in this block will cause your tests to run faster. However,
  # if you change any configuration or code from libraries loaded here, you'll
  # need to restart spork for it take effect.


  # This file is copied to spec/ when you run 'rails generate rspec:install'
  ENV["RAILS_ENV"] ||= 'test'
  require File.expand_path("../../config/environment", __FILE__)
  require 'rspec/rails'

  # Requires supporting ruby files with custom matchers and macros, etc,
  # in spec/support/ and its subdirectories.
  Dir[Rails.root.join("spec/support/**/*.rb")].each {|f| require f}

  RSpec.configure do |config|
    # == Mock Framework
    #
    # If you prefer to use mocha, flexmock or RR, uncomment the appropriate line:
    #
    # config.mock_with :mocha
    # config.mock_with :flexmock
    # config.mock_with :rr
    config.mock_with :rspec

    config.before(:suite) do
      DatabaseCleaner.strategy = :truncation
    end

    config.before(:each) do
      DatabaseCleaner.start
      #load "#{Rails.root}/db/seeds.rb"
    end

    config.after(:each) do
      DatabaseCleaner.clean
    end

  end
end

Spork.each_run do
  # This code will be run each time you run your specs.

end
{% endhighlight %}

Now start your spork server via spork command and it will let you run your rspec tests faster by providing --drb command : rspec --drb spec/
