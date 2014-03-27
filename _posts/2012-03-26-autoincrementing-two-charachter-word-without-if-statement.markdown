---
layout: post
title: "autoincrementing two charachter word without using if statement"
date: 2012-03-26 20:40
comments: true
tags: autoincrement two-charachter-word rails modulos integer-division
---

Note: This is just an interesting approach to the problem. There are special built in methods(at least in Ruby) that can do this task.
It is easy to increment just one letter but when there are more than one letter then it can be a bit challanging.
First we should inrement second letter unless it is 'Z' when it is  'Z' we should increment first letter etc. Here 
one can use 'if' statement and check if second letter is 'Z' make it 'A' and increment first letter but 
I decided to implement this without using 'if' statement. Thus my solution for this problem is with modulos and integer division operations.
In the Algebra to make domains closed people use modulos. For example let's consider this Galois field: 
{ 0, 2, 3, ..., 25 where arithmetic is performed modulo 26 } This domain is closed because of modulo 26 in other words
modulo 26 has good feature that it does not allow elements to be out of this domain. For example 2 + 26 in general arithmetic is 28 but 
with modulo 26 it is 2. So the values are repeating. In this case we do not need 'if' to say for example if result of
element incrementation is more than 25 then element is equal to 0. I've just used same logic to replace 'if statement'.
This is my function:
{% highlight ruby %}
# autoincrement code value; ie: AA, AB, AC, ..., AZ, BA, BB, ..., ZZ
def self.assign_code
  return 'AA' if count == 0 # if the collection is empty then return 'AA'
  last_code = desc(:code).first.code.to_s
  raise CodeNoUniqueValueError if last_code == 'ZZ'
  # here is how the transition done without using if statement
  last_code[0] = (last_code[0].ord + last_code[1].ord / 90).chr # if second letter is 'Z' then assign 'A' othervise increment it
  last_code[1] = ((last_code[1].ord - 64) % 26 + 65 ).chr # if second letter back to 'A' from 'Z' then increment first letter
  last_code
end
{% endhighlight %}
