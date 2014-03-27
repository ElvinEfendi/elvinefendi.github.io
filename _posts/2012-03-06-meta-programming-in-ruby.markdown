---
layout: post
title: "meta programming in ruby"
date: 2012-03-06 16:16
comments: true
categories: 
---
In this sample I will show a metaprogramming sample in Ruby. Suppose we want to add "value tracking" ability for instance variables of 
a class, that is the ability to define an instance variable in such a way that its old values can be accessible. For instance, assume we have
a Person class that has @name instance variable and if we change the value of this variable the old value should be added to history and we should 
be able to access all the old values via name\_history method. We can achieve this purpose by using metaprogramming. In this case our Person class will be as following:
``` ruby Person class
class Person
  attr_accessor_with_history :name
end
```
Now let's create attr_accessor_with_history method:

``` ruby Meta programming sample
class Class
  def attr_accessor_with_history attr_name
    attr_name = attr_name.to_s
    attr_reader attr_name
    attr_reader attr_name + "_history"
    class_eval %Q{
      def #{attr_name}= value
        @#{attr_name}_history ||= [nil]
        @#{attr_name}_history << value
        @#{attr_name} = value
      end
    }
  end
end
```

Because of in Ruby a class is an object of class Class, attr_accessor_with_history method which defined in this way will be 
accessible for all classes. In the implementation of this method metaprogramming notion is used. Because this method program the main program
, that is it adds name_history method to Person class in runtime, so it programs the program.
The other part of method is obvious; firstly it makes sure the argument is string then create getter for the instance variable, then getter for name_history after that the code inside class_eval define a setter method for instance variable and keep all its variables in the @name_history instance variable(Array type).
Now we can see the functionality:
``` ruby
filankes = Person.new
filankes.name = "Filankes"
filankes.name_history # => [ nil, "Filankes" ]
filankes.name = "Filankes1"
filankes.name_history # => [ nil, "Filankes", "Filankes1" ]
```

