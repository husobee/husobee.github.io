---
layout: post
title: "Automate those Behaviors"
date: 2015-06-13 20:00:00
categories: automate behavior testing
---

Lately I have been helping create a baseline of testing automation where I work.
Testing is hard for web application, and harder for front-ends of applications.
Here is my thought process:

* Unit Level Testing
  * Resides close to the code
  * Mocking all dependencies, we just want to test our code.
  * Written in the same language as the app

* Integration Level Testing
  * Test the service hooked up to dependencies
  * Validate that the services work together

* End User Acceptance Testing
  * Test that the end user can perform product inspired features
  * Validate that the application "works"

Interestingly enough, in thinking about this problem, I feel the integration 
testing, and the end user acceptance testing can be performed in a single pass.
It makes sense, right?  By testing "End to End" we are circumstantially testing
the integration level of the application, right?  What if we could combine these
two layers of testing, maybe into Behavior Testing?

Lucky for use, Behavior Testing is a thing, that has been around for a little 
while.  In fact, I believe [Dan North][dannorth-bdd] was the one who finally 
"got it" when realizing that requirements == behaviors == what should be tested.

I would consider the Genesis for Behavior Testing being widely adopted 
resides with the success of the [the cucumber][cucumber] project.  Cucumber 
brought to the world an English representation of a behavior in a pars-able 
syntax called [Gherkin][gherkin].

*"It is a business readable, domain specific language that lets you describe 
software's behavior without detailing how that behavior is implemented"*

As an example below, you can probably see the power of this syntax:

{% highlight gherkin %}
Feature: User's Can Register with the Service
  Registrations are the gateway to our offerings, and are a must have
  feature of the service.  Registrations will include an email address
  and a password which matches a set of rules for complexity ...

  Scenario: A User Tries to Registers with an Invalid Password
    Given a user with email address husobee@e-ie.io doesn't already exist in DB
    When a user registers with email="husobee@e-ie.io" and password="fail"
    Then the service should respond with status_code="400"
{% endhighlight %}

Wow, check that out.  When was the last time you got features so clearly 
outlined and scenarios for each feature articulated like that?  I already like
this language.

The major win, right off the bat is very clear: Testing as a concept get's put 
into the hands of the people who are responsible for testing.  QA and Product 
can clearly work together to create these features files (i.e. documentation) 
about the product.

Alright, so how does this work under the hood... can't just expect this language
to magically start doing what the English says, right?  Well each 
"Given"/"When"/"Then" step is taken, and then a map of regular expressions are
applied to it from the underlying step definitions which is in a programming 
language.

Luckily for me, there are several python variants to choose from that can read
in the Gherkin Language (the company I am at <3 python) so I started looking 
into [Lettuce][lettuce].

Below is a sample implementation of the Gherkin specified above for lettuce in 
python:

{% highlight python %}
from lettuce import step, world

@step('a user with email address ([^\s]+) doesn\'t already exist in DB')
def user_does_not_exist_in_database(step, email_address):
    # code to delete said user from database based on email_address

@step('a user registers with email="([^"]+)" and password=([^"]+)')
def user_registers(step, email_address, password):
    # code to perform a registration against the site/api
    world.registration_response = # the results from the registration
    # store the response in "world" so we can access it later
    # world is a lettuce global that is accessible from step to step

@step('the service should respond with status_code="([^"])"')
def user_registers(step, status_code):
    # check the previous step for a proper response
    assert world.registration_response.status_code == status_code

{% endhighlight %}

Now, by running lettuce the lettuce framework will run through each feature, and
each scenario in those features and regex match the corresponding python code to
run.

Major significance of this work can be boiled down to the following:

1. Behaviors are thought out and defined by the right people
2. Step code creation is very simple and straight forward
3. Regression testing can be performed by running the test suite
4. Truly, self documenting process

Major issues seen so far:

1. Frameworks out there do not scale well - Try running a huge set of tests in 
lettuce, and watch memory grow uncontrollable... i dare you. 
2. Does it count as a full "behavior" test if we are manipulating database to 
initialize each test?  Is it too clean-room to be a real test, or is setting up
the state machine good enough?

Hope this was helpful to anyone.

[dannorth-bdd]: http://dannorth.net/introducing-bdd
[gherkin]: https://github.com/cucumber/cucumber/wiki/Gherkin
[lettuce]: http://pythonhosted.org/lettuce
[cucumber]: https://cucumber.io
