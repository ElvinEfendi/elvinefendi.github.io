---
layout: post
title: "added new features to the mongoid history gem"
date: 2012-06-23 14:56
comments: true
categories: [mongoid, mongoid_history, gem, ruby, rails, track refenreced models, group history tracks]
---
Nowadays I work on a SaaS application. We develop using RoR and MongoDB. As an ODM we use Mongoid.
About two weeks ago I had a requirement to track all changes to an object and show this changes in groups. To solve this problem
I chose mongoid_history gem developed by [@dblock](http://twitter.com/#!/dblockdotorg). It is a cool gem. 
But after a while I realized this gem is not enough to do all requirements, 
because of one cannot track all referenced models with this gem([stackoverflow question regarding to this problem](http://stackoverflow.com/questions/10960124/tracking-history-of-a-model-and-all-the-associated-models-to-it-in-rails)). 
It allows to track only embedded models. That is why I decided
to extend this gem by adding new features. First of all, I will show a use case where these new features are needed, 
then I'll show how to use newly added features and after that I hope the importance of new changes will be clear.

Suppose, we have three models, called Company, Employee, Equipment. Let's define these models:
Note: I will miss unnecessary parts of models(unnecessary for my demo purpose).

Company model
{% highlight ruby %}
class Company
  include Mongoid::Document
  include Mongoid::Timestamps
  include Mongoid::History::Trackable

  # fields are defined here

  # associations
  has_many :employees, autosave: true
  has_many :equipments, autosave: true

  accept_nested_attributes_for :employees, allow_destroy: true
  accept_nested_attributes_for :equipments, allow_destroy: true

  # initialize history tracker in a same way
  track_history track_create: true,
                track_destroy: true
end
{% endhighlight %}
Employee model
{% highlight ruby %}
class Employee
  include Mongoid::Document
  include Mongoid::Timestamps
  include Mongoid::History::Trackable

  # fields are defined here

  # associations
  belongs_to :company

  # initialize history tracker in a same way
  track_history track_create: true,
                track_destroy: true

end
{% endhighlight %}
Equipment model
{% highlight ruby %}
class Equipment
  include Mongoid::Document
  include Mongoid::Timestamps
  include Mongoid::History::Trackable

  # fields are defined here
  
  # associations
  belongs_to :company

  # initialize history tracker in a same way
  track_history track_create: true,
                track_destroy: true

end
{% endhighlight %}

And suppose we are going to create, update, delete an employee, and an equipment inside the company form as a nested attributes.
In this case, if we let's say update an employee of the company then when we call company.history_tracks 
we will not get any changes or history track. This is because mongoid history gem fetches history tracks against association chain and 
in association chain it can not identify referenced objects, it can identify only embedded relations, because of an embedded object
has only one parent however when an object belongs to an object it can also belongs to another object so it is impossible to identify
its parent in this way. Therefore to solve this problem I used wrapping concept, so instead of parent child realation 
I call company a wrapper and equipment and employee wrapped objects inside company scope. This means in extended mongoid history
every object that updated, created or deleted inside a wrapper controller in our example company controller will be kept in
the collection together with wrapper object. And to fetch all the history you just need to call company.history_tracks_by_wrapper method.
This is first and I think quite important feature for history tracker. The second new feature is to group history tracks.
For example customer might ask you show changes in a groupped way, in other words when user edit a company and change some fields, add
some employees, delete some equipments etc. and then clicks update show all these changes in one group. 
A group can be defined in different way as well. Groupped history tracks can be fetched by calling company.groupped_history_tracks.
To define a group in your application you need to define history group identifier method called 'history_group_id'. This method
should be public and should return string value unique for each group. If you do not define this method as a default it will use
current timestamp with a minute precision. It means in this case if you call company.groupped_history_tracks, method will groups
history tracks by the timestamp they created. Like when you do some actions in one minute they will be groupped together and the actions in
another minute will be groupped in another group.
So as a summary these are the three methods I've added to the gem
{% highlight ruby %}
# to define groupping define this method inside wrapper controller(in the example Companies controller)
def history_group_id
  params[:group_history_by]
end

# to fetch all history by wrapper
# method of a tracked object
# format of wrapper is {class_name: '', id: ''}
company.history_tracks_by_wrapper(wrapper=nil, order_options={created_at: 'DESC'})

# fetch history tracks of the company object by wrapper(company)
# and group them by history_group_id as a default
company.groupped_history_tracks(wrapper=nil, group_by_key='history_group_id')

# to use my fork(extended mongoid_history gem in your Rails application) 
# add following liine to your gem file and rune bundle install
gem 'mongoid-history', git: 'git://github.com/ElvinEfendi/mongoid-history.git'

{% endhighlight %}

[Github url of my fork(url for extended mongoid_history gem)](https://github.com/ElvinEfendi/mongoid-history)
