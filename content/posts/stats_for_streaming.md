+++
title = "Stats for Streams 1 - Scalibility in time"
date = "2022-06-05T22:39:47+02:00"
author = "Mifour"
authorTwitter = "" #do not include @
cover = ""
tags = ["python_3", "maths"]
keywords = ["stats", "streaming", "monitoring"]
description = "Turning high school maths into efficient computer statistics"
showFullContent = false
readingTime = true
hideComments = false
+++

# Stats for Streams - part 1
*Turning high school maths into efficient computer statistics*  
  
*TLDR; The complete script is available on my [github]()*  
  
## Remembering high school maths
I guess most people did learn a few basic stats while in middle or high school.
They are fairly easy at first and are used a lot as a way to reduce a complex situation into simplier KPIs.  
  
In this article, I would like to show how to implement them in an efficient way.
Not that, the maths used here are incredible and I deserve a prize, but rather I just wanted to share an interesting idea.  
  
Let's say we have an API running on a server. It receives some requests and respond to them. 
Potentially this server can run days, weeks, months or years without ever shutingdown.
It might receive millions of requests. 
If it is used extensivly, it is probably important to some people or businesses.
To ensure it runs smoothly, we decide to measure the performances of the API, especially the response latency, for instance.  
**Just to be clear, implementing your own monitoring system is probably a wrong idea and many would suggest to use common tools. This is just a toy use case for the stats.**  

> As a side note, I would like to give some good reasons for using stats in real work.
Stats are far more interesting and useful than just having some vague insides.
With appropriated statistics, they can be used for efficient modeling and data driven decision making.  
Just an example: [Bollinger's bands](https://en.wikipedia.org/wiki/Bollinger_Bands), a financial technics from the 80s that aims to give a confidence range for the next value of a time series with a certain statistical level of 95% certainty. It is essentially computed by `[mean - 2. variance, mean+ 2.variance]`.  
  
#### Average, or mean
According to [Wikipedia Arithmetic mean](https://en.wikipedia.org/wiki/Arithmetic_mean), the non weighted mean can be computed as:  
```python
data = [0, 1, 2, 3, 4]
mean = sum(data) / len(data)
```
So, by just taking the API execution timings, it is straight forwarf to get the mean.  
But, the API justed received a new request and its execution time was 5ms.  
This starts to look like a event based value stream.  
Ok, let's just recompute the mean and update the value on the monitoriing screen, right?  
Well, we can do that but you probably feel some values are re-computed despite they did not changed.
At this point, each value adds up a bit of weight on our computation as well our memory impact. 
That's not good. It will definitly create some scalabilty issues later. We are creating new issues that are not even related to the core business of the API.  
  
There is many ways to resolve this issue.  
I would say, a good approach (and probably the most realistic one to be used in real world) is to limit the maximum amount of data used for the computation. 
Like, override the latest value when is more than 100 values, by using a fixed sized array.
Or, let the values expire after a certain amount of time. This would some kind of sliding window, a technic used in real world applications.  
But, what if we do not have to limit our scope?  
Can we do better?  
  
## Rewriting the Average function
Updating the mean value can be done like this:
```python
data = [0, 1, 2, 3, 4]
nb = len(data)
total = sum(data)
mean = total / nb
print(f"{mean=}")
new_value = 5
total += new_value
nb += 1
mean = total / nb
print(f"{mean=}")
```
Like this, no computation was done twice.
That's much better.
Let's turn that into a function.
Because we want to release the execution to other computations while waiting for new values, we can take advantage of Python's generators direclty.
```python
from random import randint


def mean(buffer):
    total = 0
    nb = 0
    while True:
        for data in buffer:
             total += data
             nb += 1
        yield total / nb


buffer = []
# beware to keep the same buffer pointer all the time
# by using no redefinition instructions
mean_gen = mean(buffer)
for _ in range(100):
    buffer += [randint(0, 10) for __ in range(3)]
    print(f"avg: {next(mean_gen)}")
    buffer.clear()
```

#### DRYing the code
Cool. Now let's do the same to find the maximum execution time.
```python
def maximum(buffer):
    current = float("-inf")
    while True:
        for data in buffer:
             current = max(data, current)
        yield current


buffer = []
max_gen = maximum(buffer)
for _ in range(100):
    buffer += [randint(0, 100) for __ in range(3)]
    print(f"max: {next(max_gen)}")
    buffer.clear()
```
It looks redundant to me. Surely we can DRY our code and abstract the generator logic from the math logic.  
Now the whole script is:
```python
from random import randint


def streaming_func(func, buffer, *args):
	"""
	 Using only scalars within lambda ensure the scalability to be:
	 O(1) time and space for each new value
	"""
	while True:
		for data in buffer:
			res, args = func(data, *args)
		yield res

"""
Find the maximum of a stream
"""
maximun = lambda data, current_min: (
	data if data > current_min else current_min,
	(data if data > current_min else current_min,)
)

liste = [randint(0, 100) for _ in range(100)]
classic_min = min(liste)

buffer = []
buffer_size = 3
gen_max = streaming_func(maximun, buffer, float("-inf"))
for index in range(0, 100, buffer_size):
	buffer += liste[index: index + buffer_size]
	stream_max = next(gen_max)
	buffer.clear()

print(f"{stream_max=}")

"""
Find the average of a stream
"""
avg = lambda elem, mean, nb: (
	(mean * nb + elem) / (nb + 1),
	(
		(mean * nb + elem) / (nb + 1),
		nb+1
	)
)

liste = [randint(0, 100) for _ in range(100)]
classic_avg = sum(liste) / 100

buffer = []
buffer_size = 3
gen_avg = streaming_func(avg, buffer, 0, 0)
for index in range(0, 100, buffer_size):
	buffer += liste[index: index + buffer_size]
	stream_avg = next(gen_avg)
	buffer.clear()

print(f"{stream_avg=}")
```
I won't explain much of this code but if you feel you need undertstand a bit more, here's essentially what I did:  
I used a higher order generator to loop infinitly over a callable (those are my lambda functions) with a arbitrary number of arguments. The Lambda function return the result and the updated arguments. Then the generator yields the result.  

## Now, Let's get the Standard Deviation function
In our use case, we might be interessed to know tightly our API can respond in a predictable maneer.
Maybe there is another network call of other process that block the incoming requests sometimes.  
Intuively, we can think of the standard deviation like an indication of the distance between values and the mean value.    
  
According to the [wikipedia Standard Deviation](https://en.wikipedia.org/wiki/Standard_deviation), we can compute the stddev like this:
```python
data = [0, 1, 2, 3, 4]
mean = sum(data) / len(data)
stddev = sum((value - mean)**2 for value in data) / len(data)
```
But we already know how to compute the average in streaming fashion.
The rest of the equation only depends on the current value, we're already almost there.  
  
#### A bit of maths
Grabing a pen and a sheat of paper is certainly a good idea at this point.  
With &mu; being the mean value and I the number of processed values.


var = <MATH>&sum;(x_i - &mu;)² / I</MATH>  
<MATH>
var = ((x_0 - &mu;)² + (x_1 - &mu;)² + (x_2 - &mu;)² + ... ) / I
</MATH>

You probably remember your middle school maths:

<MATH>(a - b)² = a² - 2.a.b + b²</MATH>

Thus:  

var = <MATH>(x_0² - 2.x_0.&mu; + &mu;² + x_1² - 2.x_1.&mu; + &mu;² + x_2² - 2.x_2.&mu; + &mu;² + ...) / I</MATH>  
var = <MATH>(&sum;x_i² - 2.&mu;.&sum;x_i + I.&mu;² )/ I</MATH>

#### Back into code

Now, it is possible to compute the stddev by processing one and only one time each value. We always get a correct stddev value by updating variables.
In code, it looks like this:
```python
from numpy import std  # just to assert we got the correct value

stddev = lambda data, mean, nb, total_sum, total_square: (
	((total_square+data**2) - 2*mean*(total_sum+data) + (nb+1) * mean**2 ) / (nb+1),
	(
		(mean*nb + data)/(nb + 1),  # mean
		nb + 1,  # nb of values
		total_sum + data,  # sum of values
		total_square + data**2  # sum of squared values
	)
)

liste = [randint(0, 100) for _ in range(100)]
classic_stddev = std(liste)

buffer = []
buffer_size = 5
gen_var = streaming_func(stddev, buffer, 0, 0, 0, 0)
for index in range(0, 100, buffer_size):
	buffer += liste[index: index + buffer_size]
	stream_variance = next(gen_var)
	buffer.clear()

assert round(classic_stddev, 1) == round(stream_variance**0.5, 1)
# the repeating division may create rounding error

```
We did it! It wasn't that hard.
Nonetheless, I find remarkable the possibility to do turn a formula that need to process all the values every single time into one that processes values only one time.
What a save in compute!  

## What about Median value?
Mean value is good but median value is better.  

Helas, it's a this point I admit the median value, that I would love to compute the same way, has a messier computation.  
By [wikipedia definition](https://en.wikipedia.org/wiki/Median)  
> the median can be defined as follows: For a data set  x of n elements, ordered from smallest to greatest,  
  
This prevent us to update the median value in constant time, because it requires to keep track of all values all the time.  
Too bad.  
  
But, that doesn't mean it not possible to have a fast implmentation. By separating the update from the final compute of the value...
This limits the set of unique values vastly smaller than the whole collection of values. A small set might be a discret measure with a small cardinality ie the marks of the students (A, B, C, D, E). A continous space of possible values would not scale, like the gas price or the height of a population... Also, the median does require to have comparable values (any that can be represented by numeric values). There would be no sense to talk about the median color of the set of cars (red, grey, yellow...).  
  
```python
from collections import Counter

class Percentile():
	nb = 0
	counter = Counter()
	tree = Tree()

	def update(self, data):
		nb += 1
		self.counter.add(data)
		if data not in self.tree:
			self.tree.add(data)

	def compute(self, percent):
		assert 0 <= percent <= 100
		actual_percent = 0
		while actual_percent < percent:
			value = next(self.tree)
			actual_percent += (self.counter[value] / self.nb) * 100
		return value

```
This implementation tend to scale in O(N + k.log(k) + log(k)), with k being the size of the set of unique values. If k is small enought compare to N, the number of values, this implementation will be fast.      

## Bonus, any callable works
Earlier I used `lambda` functions as callables inside my higher order generator.
But really, any callable would just work.  
  
Here's some alerting system for the API (probably the worst alerting system there is)
```python
msg = "INFO: value is: "
def alerting(data, _):
	if data > 90:
		print(msg + str(data))
	return None, [None]	

liste = [randint(0, 100) for _ in range(100)]

buffer = []
buffer_size = 3
gen_alert = streaming_func(alerting, buffer, [])
for index in range(0, 100, buffer_size):
	buffer.clear()
	buffer += liste[index: index + buffer_size]
	_ = next(gen_alert)

```

## next
Thanks to this implementation, things can keep getting higher and faster.  
[Here's part 2](http://localhost:1313/posts/stats_for_streaming_part2/) that involves Rust and parallele computing.
_________________
## Thanks

Thank you for reading this article. I hope you like it and felt the mathematical wonder about turning math formula into an efficient implementation.  
  
Have a nice day,  
Mifour :) 
