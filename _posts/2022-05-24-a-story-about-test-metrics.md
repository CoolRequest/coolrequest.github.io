---
layout: post
title: A Story about Test Coverage Metrics
tags: testing
date: 2022-05-24
---

Once upon a time, there was a Ruby on Rails programmer called John. John cared about testing his code very thoroughly.

He had recently learned about automatic code coverage analysis, and was very excited to know that not only you can write automated tests, but you can also ensure that 100% of your code is covered by them. This way nothing could go wrong!

So, one day, his boss told him about the requirements for a new software. He needed to calculate the market price of a house, using the following specification:

<div class="message">
<p>A house is considered big if it has at least <b>3 rooms</b> and <b>2 bathrooms.</b> </p>

For big houses, use this formula to calculate the price:<br>

<pre>$1000 + 100 * (number_of_rooms + number_of_bathrooms)</pre>

And for small ones, use this instead: <br>

<pre>$500 + 100 * (number_of_rooms + number_of_bathrooms)</pre>
</div>

That's pretty easy, John thought. So he coded the function below:

{% highlight ruby %}
def house_price(rooms, bathrooms)
  if rooms >= 3 && bathrooms >= 3
    1000 + 100 * (rooms + bathrooms)
  else
    500 + 100 * (rooms + bathrooms)
  end
end
{% endhighlight %}

And, of course, the unit tests:

{% highlight ruby %}
RSpec.describe 'calculate house price', type: :model do

  it 'small house with 1 room and 1 bathroom' do
    price = house_price(1, 1)

    expect(price).to eq 700 # 500 + 100 * (1 + 1)
  end

  it 'big house with 4 rooms and 4 bathrooms' do
    price = house_price(4, 4)

    expect(price).to eq 1800 # 1000 + 100 * (4 + 4)
  end
  
end
{% endhighlight %}

Test coverage, as calculated by simplecov, was at 100%. The system went into production and everyone was happy.

Until one day, John's boss complained to him that he had found a bug. The system was incorrectly calculating the price for a house: <br>

{% highlight ruby %}
Rooms: 3
Bathrooms: 2
--> Calculated price: 1.000
{% endhighlight %}

That should be $ 1.500, the boss said. And he was right, of course.

John could not believe it. He had 100% test coverage! How could there still be a bug in his code?

He checked his code again and found the bug:

{% highlight ruby %}
(...)
  # clearly wrong:
  if rooms >= 3 && bathrooms >= 3
(...)
{% endhighlight %}

So what is the message here? Why was 100% test coverage not enough? *He should have tested the edge cases!*, you might be thinking. If the boundary of the condition is 2 bathrooms, he should have test cases with 1, 2 and 3 bathrooms.

Although that <i>is</i> correct, I believe that this simple example reveals a deeper concept. When we measure test coverage by lines of code, we are ensuring that all parts (or at least most parts) of our code somehow gets executed during the tests. That is good, but far from enough.

If you think of your program as a machine that transforms inputs into outputs, you should be considering test coverage on terms of the input space. Think about:
- What are the inputs?
- What value can each of this inputs take?
- How can they be combined?
- Am I testing all the combinations?

Of course, most of the time we are talking about strings, integers, and floating point numbers, and and the possible combination of inputs is huge, making it unfeasible to test them all. 
But you shoud be able to partition this space into meaningful subsets.

Back to our example, one possible way to partition the input space is this:

<img src="/public/images/test_metrics_1.png">

To keep things simple, we are not even separating edge cases or invalid inputs, which should ideally be considered as separate test cases. But even this way, as shown on the diagram, there should be at least 4 test cases.

Thinking about your tests in this perspective can also help identify if your code is not covering all the possibilities. Imagine, in our example, if the programmer had written this code:
{% highlight ruby %}
def house_price(rooms, bathrooms)
  if rooms >= 3
    1000 + 100 * (rooms + bathrooms)
  else
    500 + 100 * (rooms + bathrooms)
  end
end
{% endhighlight %}

It is obviously missing a conditional, wright? And it still passes all the tests that John wrote, with 100% coverage. However, if you create one test on each of the areas of the diagram above, you will find out that it fails test case #2 (>= 3 rooms, < 2 bathrooms).

This simple story reveals one important limitation of the metric of "lines of code" coverage metric: it will tell you nothing about what percentage <i>of the possible inputs</i> you have covered. 

There are many other limitations. One of them deserves to be mentioned: your assertions are crucial to the effectiveness of your tests. If you have good code coverage, but silly assertions, you are only checking if your code doesn't raise exceptions, not if it produces the desired results. For example:
{% highlight ruby %}
def my_function
  do_this_thing()
  do_that_thing()
end

def my_function_test
  assert this_thing_done()
end
{% endhighlight %}

The lines of code inside <b>do_that_thing()</b> are covered by tests, but the test will always pass, even if the result is wrong (assuming it doesn't raise an exception, of course).

So, what does all of that mean? I am not saying that you should stop measuring code coverage. It has many benefits. Just keep these points in mind:
- Having 100% code coverage is a good thing, but does not guarante that your tests are really effective
- When designing your tests, think about what are the requirements, inputs and outputs. Do you have tests for all the scenarios?
- Make sure the assertions are really checking if the output is correct


At the end of the day, if you want to assess the completeness of your test suite, you still need a human being to analyze it. Tools will help, but they will not tell you the complete story.