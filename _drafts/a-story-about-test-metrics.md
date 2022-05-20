---
layout: post
title: A Story about Test Coverage Metrics
---

Once upon a time, there was a Ruby on Rails programmer called John. John cared about testing his code very thoroughly.

He had recently learned about automatic code coverage analysis. He was very excited to know that not only you can write automated tests, but you can also ensure that 100% of your code is covered by them. This way nothing could go wrong!

So, one day, John's boss told him aboout the requirements for a new software. He needed to calculate the sale price of a house, using the following specification:

<div class="message">
A house is considered big if it has at least 3 rooms and 2 bathrooms. <br>

For big houses, use this formula to calculate the price:<br>

$1000 + 100 * (number_of_rooms + number_of_bathrooms)<br>

And for small ones, use this instead: <br>

$500 + 100 * (number_of_rooms + number_of_bathrooms)
</div>

That's pretty easy, John thought. So he coded the function below:

{% highlight ruby %}
def calc_house_value(rooms, bathrooms)
  if rooms >= 3 && bathrooms >= 3
    1000 + 100 * (rooms + bathrooms)
  else
    500 + 100 * (rooms + bathrooms)
  end
end
{% endhighlight %}

And, of course, the unit tests:

{% highlight ruby %}
RSpec.describe 'calc value', type: :model do
  it 'caso 1' do
    calc_house_value(1, 1)
  end

  it 'caso 2' do
    calc_house_value(4, 3)
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

So what is the message here? *He should have tested the edge cases!*, you might be thinking. Altough that is correct, I believe that this simple example reveals a deeper concept.

When we measure test coverage by lines of code, we are ensuring that all parts (or at least most of) of our code somehow gets executed during the tests. That is good, but far from enough.

If you think of your program as a machine that transforms inputs into outputs, you should be considering test coverage on terms of the input space. What are the different combinations of inputs that are possible? Am I testing them all?

Of course, most of the time, we are talking about strings, integers, and floating point numbers, and the possible combination of inputs is huge. But you shoud be able to partition this space into meaningful subsets.

Back to our example, one possible way to partition the input space is this:

<table>
  <thead>
    <tr>
      <th>Case #</th>
      <th>Rooms</th>
      <th>Bathrooms</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>< 3</td>
      <td>< 2</td>
    </tr>
    <tr>
      <td>2</td>
      <td>>= 3</td>
      <td>< 2</td>
    </tr>
    <tr>
      <td>3</td>
      <td>< 3</td>
      <td>>= 2</td>
    </tr>
    <tr>
      <td>4</td>
      <td>>= 3</td>
      <td>>= 2</td>
    </tr>
  </tbody>
</table>

<div style="color: red;">
  ** Essa tabela bem que poderia ser uma figura bonita com um plano cartesiano dividindo o espaço em 4 quadrantes
</div>

To keep things simple, we are not even separating edge cases or invalid inputs, which should ideally be considered as separate test cases. But even this way, if you look carefully, there should be 4 test cases.

This story reveals one limitation of the metric of "code line coverage".
There are many others. For example:, if the assertions in your test do not check for all the outputs, your test can be useless even though the code coverage is good. For example:
```
def my_function
  do_this()
  do_that()
end

def my_function_test
  assert(this.done())
end
```

And, of course, if our tests are not in sync with your requirements, that can be a problem. It is easy to spot it when you add new requirements - because if new tests are not created, the test coverage will drop. But what happens when some requirement is removed? You are stuck with useless or unreachable code in your application, and the test coverage metric says everything is great!

Bottomline:
- Having 100% code coverage is a good thing, but does not guarante that your tests are really effective
- It is important to understand what is not covered by that 100%
- Be careful about the assertions in your test
- When you design your tests, try to think in terms of what are the possible inputs. You will probably come up with test scenarios that you might otherwise have missed.