---
layout: post
title:  "a tip regarding to mongoid history modifier field"
date:   2012-06-23 16:35
comments: true
tags: [mongoid history, mongoid, rails, modifier field does not get set]
---

If you are reading this post you are probably a user of mongoid history gem. So most likely you have faced an issue
that modifier field does not get set. It simply becomes nil. This is because in development mode Rails does not load
all models/classes when server starts. In this case because of HistoryTracker model is not loaded filter does not get added to the
appropriate controller therefore it can not get current_user method. To solve this issue just add following line in your 
mongoid_history initializer:
{% highlight ruby %}
  Mongoid::History.tracker_class_name = :history_tracker
  Mongoid::History.current_user_method = :current_user
 
 # in development mode rails does not load all models/classes
 # force rails to load history tracker class
 Rails.env == 'development' and require_dependency(Mongoid::History.tracker_class_name.to_s)
 end
{% endhighlight %}
