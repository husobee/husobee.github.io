---
layout: post
title: "Never Use Floats for Money"
date: 2016-09-23 05:00:00
categories: money float
---

### Very basic background

Humans count and perform math in base ten or denary.  This means there are ten
digits to represent all numbers and the base number is ten.  Thousands are 10^3,
Hundreds are 10^2 and so on.  We have adopted denary, probably, due to the fact 
that most of us have 10 fingers, and learned to count on our fingers.  Moreover
the word digit is a bi-word for finger.

Computers count and perform math in base two or binary.  This means there are 2
digits to represent all numbers.  2^3 is the fourth position, 2^2 is the third 
position and so on.  110 is 1 * 2^2 + 1 * 2^1 + 0 * 2^1 = 6 in denary.

### Onward

Lets say we want to divide 1/3 and represent that in denary.  We would have to
represent that in a forever repeating decimal 0.333333... which is only ever
going to be a representation of the true value of 1/3.  Now if we were using 
ternary, or base three number representation 1/3 would actually be represented
as exactly 0.1, as the first decimal position is 3^-1 which is the same as 1/3.

So in our denary system we have an approximation of 1/3 but in ternary we have
an exact representation of .1.

This is precisely the problem we have when trying to represent 10^-1, or 0.1 in 
binary.  There is not an exact binary representation of 0.1 or 0.01.

### Why floats are bad for money

We would think yeah, floats would be great for money, because $1.40 is 1 dollar
and 4 tenths of a dollar. There is a decimal point, to have decimal points in 
code we would have to use a double or floating point type.  Well floating point
types are actually binary representations of denary numbers as mentioned above.

The float type has a sign, exponent, and fraction blocks within the 32 or 64 bit
structure which can be seen [on wikipedia][float-wiki].  You can see that you 
have at your disposal a 23 bit binary fraction and an 8 bit binary exponent.

Example:

I sell 165 apples at $1.40 per apple.  My accounting software uses floating point
numbers for the calculation.

{% highlight text %}
>>> 165 * 1.40
230.99999999999997
{% endhighlight %}

As you can see in this example, you would need to perform rounding yourself to
get the correct number.  Here is another common example, you are changing the 
the price of bananas from $1.01 to $0.99 and need to calculate the lost revenue.

{% highlight text %}
>>> 1.01 - 0.99
0.020000000000000018
{% endhighlight %}

Again you can see the that a rounding function would need to be applied.  Moreover,
if you start getting into really big numbers you start loosing granularity, and 
the smaller numbers start getting eaten by the bigger numbers:

{% highlight text %}
>>> 1e16 * (1.01 - 0.99) - 0.01
200000000000000.2
{% endhighlight %}

The above, in an exact denary calculation should be 199999999999999.99.

The more operations you perform the worse your estimation becomes.

### Another option

Another option is to use plain integers to represent currency.  If you make $1.01
into 101¢ and perform all your calculations on cent integers you will never run
into these floating point approximation issues, as you are no longer performing
approximations.  Here are the above examples using integer cents instead:

{% highlight text %}
>>> 165 * 140
23100
>>> 101 - 99
2
{% endhighlight %}

There is also a good explanation of [different solutions here as well][money-so]


[float-wiki]: https://en.wikipedia.org/wiki/Single-precision_floating-point_format
[money-so]: http://stackoverflow.com/questions/3730019/why-not-use-double-or-float-to-represent-currency#3730040
