---
layout: post
title: "hierarchical user authorisation in rails using cancan"
date: 2012-03-13 19:31
comments: true
tags: cancan user hierarchy authorisation
---
Assume that we need a web application that should have users with different roles. Furthermore let's say we need these roles 
to be hierarchic in other word suppose we have Admin, Manager, Seller, Buyer and Reporter roles. 
Admin can do everything that Manager, Seller, Buyer can, Manager can do everything that Seller and Buyer can. Seller, Buyer and Reporter
are the last elements of this hierarchy.
So there is an hierarchy between roles. Lets implement this authorisation system. 
I will use Mongoid ODM in my models. First we need to create User model as follows:

{% highlight ruby %}
class User
  include Mongoid::Document

  field :name, type: String
  field :mobile, type: String
  field :phone, type: String
  field :address, type: String

  belongs_to :role

  def role?(base_role)
    self.role ||= Role.default
    role.is_parent_of? base_role
  end

end
{% endhighlight %}

Now let's define Role model

{% highlight ruby %}
class Role
  include Mongoid::Document
  
  field :name, type: String
  field :parent_id, type: BSON::ObjectId

  # validations
  validates_uniqueness_of :name
  
  has_many :users

  def is_parent_of? base_role
    while base_role && base_role != self
      begin
        base_role = Role.find base_role.parent_id || 0
      rescue Mongoid::Errors::DocumentNotFound => e
        base_role = nil
      end
    end

    base_role == self
  end

  def parent= parent_role
    self.parent_id = parent_role.id
  end

  def self.method_missing method_id
    where( name: method_id ).last # allow us get a role as Role.admin etc.
  end

  # return default role
  def self.default
    Role.buyer
  end

end
{% endhighlight %}

Defining User model and Role model in this way will allow us to make role hierarchical. That is, in the console we can try following:
Before trying in console create sample models and users.

{% highlight ruby %}
admin = User.find <admin user id>
admin.role? Role.admin # => True
admin.role? Role.manager # => True
manager = User.find <manager user id>
manager.role? Role.seller # => True
manager.role? Role.admin # => False
{% endhighlight %}

Now we have general relationship logic between roles that can be used either for creation hierarchical or non hierarchical roles.
It is time to create abilities. I suppose you can use CanCan. First we need to create Ability model to define abilities.
To do this CanCan has generator like this: rails g cancan:ability. After running this command edit the Ability model and define abilities 
according to your application's business logic. Some samples:

{% highlight ruby %}
class Ability
  include CanCan::Ability

  def initialize(user)
    user ||= User.new

    if user.role? Role.buyer
      can :read, Product
    end

    if user.role? Role.seller
      can :update, Product
    end

    if user.role? Role.manager
      can :manage, User
    end

    if user.role? Role.reporter
      can :manager, Report
    end
    
    # See the wiki for details: https://github.com/ryanb/cancan/wiki/Defining-Abilities
  end
end
{% endhighlight %}

In this case because of we have hierarchy so that admin is also manager admin will be able to read, update Product, manager User and Report.
But reporter wil only be able to manage reports.
