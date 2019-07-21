---
layout: post
title:  "Fixing Flaky Tests Like a Detective (RailsConf 2019)"
date:   2019-07-20 18:42:49 -0500
tags: ["Ruby", "Ruby on Rails", "Testing"]
image: "/assets/images/posts/flaky_tests.png"
---

This spring I spoke at RailsConf about fixing flaky tests in Rails, and how reading lots of mystery novels helped me learn how to do it. Here's the video of my talk - if you prefer to read about it instead, I've also adapted it into a blog post below.

<iframe width="560" height="315" src="https://www.youtube.com/embed/qTyoMg_rmrQ" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<h3>Table of Contents</h3>
* toc
{:toc}

## My first encounter with a flaky test

I want to start out by telling you a story. It was my first year as a software engineer, and I’d worked really hard building out a complicated form - my first big frontend feature. I wrote lots of unit and feature tests to make sure I didn’t miss any edge cases. Everything was working great, and we shipped it! But then, a few days later, it started.

A red build on our master branch. The failing test was one of the feature tests for my form. Huh?? It had been working fine for days!

The first time it came up, we all kind of ignored it. Tests fail randomly once in a while, and that's fine, right?

<figure style="text-align: center;">
  <img src="https://media.giphy.com/media/QZkpIdieotn3i/giphy.gif" alt="Ignoring a computer on fire">
  <figcaption class="caption-text">This is fine.</figcaption>
</figure>

But then it happened again. And again.

I thought, no problem, I’ll spend an afternoon digging into it and fix it.

The only problem was: I had never fixed a flaky test before and had no concept of why a test would pass or fail on different runs. 

So I did what I often did when debugging problems I didn’t really understand at all… I tried to use trial and error: making a guess at a random change without really understanding how it would help.

That kind of approach sometimes worked with normal bugs - you could start by using trial and error, quickly ruling out various potential solutions, and eventually you'd hit on the right one, at which point you could reason backwards about why it worked. But that didn't help at all with this flaky test, because I could never be sure if my random fixes worked. I'd run it with the fix 50 times without seeing a failure and think I solved it, but the next day we'd see it fail agian. 

And that’s exactly what makes fixing flaky tests so challenging. Trying random fixes and testing them by running the test over and over to see if it fails doesn’t work well and is a very slow feedback loop.

<figure style="text-align: center;">
  <img src="https://media.giphy.com/media/AVx9avUHC0i9a/giphy.gif" alt="Corgi running in circles">
  <figcaption class="caption-text">Trying to fix a flaky test with trial and error</figcaption>
</figure>

We eventually figured out a fix for that flaky test, but not until several different people had tried random fixes that failed and it had sucked up entire days of work.

The other thing I learned around that time is that even just a few flaky tests can really slow down your team. When a test fails without actually signaling something wrong with the test suite, you not only have to re-run all the tests before you’re ready to deploy your code, which slows down the deployment process,

You also lose some trust in your test suite, and eventually might even start ignoring real failures because you assume they’re just flakes.

So it’s super important to learn how to fix flaky tests efficiently, and better yet avoid writing them in the first place.

## A better method

For me, the real breakthrough in fixing flaky tests was when I came up with a method to follow. 

1. I gathered all the information I could find about the flaky test and the times that it had failed. 
2. I tried to fit it into one of 5 main categories of flaky test (more on those in a moment).
3. Based on that, I came up with a theory of how it might be happening. 
4. Then, I implemented my fix.

At the same time that I was figuring this out, I was on kind of mystery novel binge. And it struck me that every time I was working on fixing a flaky test, I felt kind of like a detective solving a mystery. After all, the steps to do that (at least in the novels I read, which are probably very different than real life) are basically:

1. You start with gathering evidence.
2. You identify suspects.
3. You come up with a theory of means and motive.
4. And then you solve it!
   
Thinking about it that way made fixing flaky tests more enjoyable and it actually became a fun challenge for me instead of a frustrating and tedious problem.

## Gathering evidence

There are lots of pieces of information that can be helpful to have when you’re trying to diagnose and fix a flaky test. They include:

* Error messages and output from each failure
* Time of day failures occurred
* How often the test is failing
* Which tests were run before, in what order

A method that I’ve used to efficiently collect this information in the past is to have any time a test fails on your master branch automatically sent to a bug tracker with all the metadata you’d need, including a link to the CI build where they failed. I’ve had success doing this with [Rollbar](https://rollbar.com/) in the past, but I’m sure other bug trackers would work as well. 

When doing that, it’s important to make sure the failures for the same test can generally be grouped together - it might take a little configuration or finessing to get this to work, but it’s really helpful so that you can cross reference between different occurrences of the same failure and figure out what they might have in common.

## Identifying suspects

Now that you have your evidence, you can start looking for suspects. With flaky tests, the nice thing is that there’s basically always the same set of usual suspects to start with, and then you can narrow down from there. Those suspects are:

* Async code
* Order dependency
* Time
* Unordered collections
* Randomness

### Suspect #1: Async code

Async code is, in my experience, one of the biggest sources of flaky tests, especially when testing Rails apps.

When I say “async code”, I’m referring to tests in which some code runs asynchronously, which means that events in the test can happen in more than one order

The most common way this comes up when testing Rails apps is in your system and feature tests. Most Rails apps use Capybara, either through Rails’ built in system tests, or RSpec feature tests, to write end to end tests for their application that spin up a Rails server and a browser, and interact with the app like a user would.

The reason you’re necessarily dealing with async code and concurrency when you write Capybara tests is that there at least 3 different threads involved: 

* The main thread executing your test code
* Another thread spun off to run the server for your Rails app
* A thread in a separate process running the browser, which Capybara interacts with using a driver

#### Examples

Let’s look at how this plays out in a simple example. Imagine you have a Capybara test that clicks on a Submit Post button, and then checks that a post was created in the database.

``` ruby
click_on "Submit Post"

expect(Post.count).to eq 1
```

Here’s what the happy path for this test looks like:
1. The test code sends a message to the browser to trigger a click event on the button
2. In the browser, that click triggers an ajax request to the Rails server
3. The rails server processes that request and sends the response back to the browser.
4. The browser gets the response and updates the UI.
5. The test code checks the db and finds the newly created post. 

But, that happy path isn't the only order events in the test. Since we’re not explicitly waiting on anything that would signal that the request to create the post has finished, our test code can move right along from #1, tirggering the click, to #4, checking the database, before steps 2, 3 and 4 have finished.

The fix here is relatively simple - we just need to make sure that we wait until the request has finished before we try to check the database. We can do this by adding one of Capybara’s waiting finders, like `have_content`, which will look for something on the page and retry until it shows up.

``` ruby
click_on "Submit Post"

expect(page).to have_content("Post Created")

expect(Post.count).to eq 1
```

That’s a relatively simple async flake. But they can get a lot more complicated. Let's take a look at a trickier one.

``` ruby
visit books_path
click_on "Sort"
expect_alphabetical_order

click_on "Sort"
expect_reversed_alphabetical_order
```

Let's say that `expect_alphabetical_order` and `expect_reversed_alphabetical_order` are custom helpers we've implemented that use Capybara's waiting finders under the hood to wait until the books show up in that particular order. So this seems like it should work: we're waiting between each action and even if there's a delay, Capybara should just retry until the correct order shows up. But then this starts to flake.

How does this happen? Consider the situation where we land on the books page, and the books just so happen to already be in the correct order. Then: 
1. We click on 'Sort' 
2. `expect_alphabetical_order` passes immediately
3. We move on to click on 'Sort' again, but _before_ the page has even started reloading with the books from the first click on "Sort". 

This results in basically a "doubleclick" effect, rather than the two separate page reloads we want, so the page just reloads once with the books in the ascending alphabetical order, and we’ll never get to that reversed alphabetical order state.

The solution here would be to wait on something besides the fact that the books are in the correct order to make sure that the first request has completed before we try to trigger the second one. Here's an example of how the fixed test might look:

``` ruby
click_on "Sort"
expect(page).to have_content("Sort: ASC")
expect_alphabetical_order

click_on "Sort"
expect(page).to have_content("Sort: DESC")
expect_reversed_alphabetical_order
```

#### Identifying async flakes

* **Is it a system or feature test** (AKA, some kind of test that interacts with your app via the browser?). If it is, look for any events in it where you aren't waiting for the result before moving on.
* **Capture some screenshots**: With most Capybara drivers, you're able to call `save_screenshot` at any point to save a screenshot of the page, usually both as html and a png. You can also use the [capybara-screenshot](https://github.com/mattheworiordan/capybara-screenshot) gem, which helps you ensure that any time a test fails, a screenshot is saved. 

#### Preventing async flakes

* **Make sure your test is waiting for each action to finish**: break your test down into individual actions, and check that in between each of them, you're waiting for a clear sign that the results of that action are complete.
* **Don’t use `sleep` - wait for something specific**: using `sleep` is unreliable & slower than waiting for a specific event
* **Understand which Capybara methods wait & retry, and which don’t**: Make sure you’re familiar with Capybara’s API and know which of its methods will wait. Everything based off “find” will wait, but certain things, like “all”, will not wait.
* **Check that each assertion is working as expected**: the alphabetical order was a great example of a situation where it looked like we were doing everything right, but because things weren't happening under the hood the way we thought they were, we ended up with a flake.

### Suspect #2: Order dependency

**Definition**: Order dependent tests are those that can pass or fail depending on which tests ran before them. 

Usually, this is caused by some sort of state leaking between tests - so when the state another test creates is present (or not present) it can cause the flaky test to fail.

#### Sources of shared state

Shared state can sneak into your test suite in several different ways, including:
  * database state 
  * global and class variables in your app
  * browser state, for system & feature tests that use a browser.


#### Order dependency caused by database state

When you're writing tests, each test should start with a "clean" DB - that might not mean with a fully empty database, but if anything is created, updated or deleted in the database DURING a single test, it should be put back the way it was at the beginning. This is important because otherwise, those changes to the database could have unexpected impacts on later tests, or create dependencies between tests so you can't remove or re-order tests without risking cascading failures.

Database cleaning in Rails should "just work" and often does, especially if you're able to use Rails' basic, built in transactional cleaning. But there are so many different ways you might have your Rails test suite configured, and it _is_ possible to do it in such a way that certain gotchas are introduced. So it's important to know how your database cleaner works, when it runs, and if there's anything it's leaving behind - especially if you're dealing with a flaky test that seems to be order dependent.

#### Ways to clear your database state

* **Transactions**
  * Wrapping your test in a transaction & rolling it back afterwards is generally the fastest way to clear your database, and it's the default for Rails tests 
  * In the past you couldn’t use it with Capybara tests because the test code and test server didn’t share a database connection, but Rails 5 system tests addressed that issue by allowing the test code and test server to have shared access to a database connection during the test
  * This approach still requires you to be careful about a couple of things. Transactions might have slightly different behavior than you see in your actual app - for example, if you have any after_commit hooks set up on your models, they won’t be run.
* **Truncation or deletion**
  * Because of the limitations of transactional cleanup, many Rails test suites use the [database_cleaner](https://github.com/DatabaseCleaner/database_cleaner) gem to clean with either truncation - truncating all tables touched in the test - or deletion, which uses DELETE FROM to delete all records created in a test. 
  * Since truncation and deletion let you run the app just like it normally would — without extra transactions wrapping each test — they’re a little more realistic and that can make them easier to work with. 
  * If you use database_cleaner, you should also make sure you’re running it in an `append_after(:each)` block, not an `after(:each)` block- this ensures it runs after Capybara’s cleanup, which includes waiting for any lingering ajax requests that could affect the state of the database and need to be cleaned up.

#### Example

Let's say we're using `database_cleaner` with the truncation strategy:

``` ruby
DatabaseCleaner.strategy = :truncation
```

But we notice that this is slow, so someone comes along to optimize the test suite. They notice we're creating book genres in almost all of the tests, so they decide to do that before the entire suite runs instead using a `before(:all)` block, and then exclude those from being cleaned up with `database_cleaner`, with the built in option for that:

``` ruby
DatabaseCleaner.strategy = :truncation, {:except => %w[book_genres]}
```

This saves us some time, but introduces a gap in our cleaning. If we make any kind of modification to BookGenre, that update won't be undone, because we'll never truncate those tables. And this could potentially affect later tests, and show up as an order dependent flake. 

``` ruby
book_genre = BookGenre.find_by(name: "Mystery")
book_genre.update!(status: "deleted")
```

To be clear, I'm not picking on `database_cleaner` here - I'm just giving an example of how a minor configuration change could allow you to create more flakes, and why it's important to therefore have a good understanding of how cleaning is actually working and the trade-offs you introduce depending on how you do it.

#### Other sources of order dependency
* **The browser**:  since tests run within the same browser, that can contain specific state depending on the test that just run. Capybara works hard to clean this all up before it moves on to the next test, so this usually should be taken care of for you, but it's likely possible that depending on your configuration you could have some issues, so it's good to be aware of that as a possibility.
* **Global/class variables**:  if you modify those, that can persist from one test to the next. Normally Ruby will yell at you if you reassign these, but one area where they can sneak through is if you have a hash and you just change one of the values in it - since that isn't reassigning the entire variable, it won't come up as a warning.

#### Identifying order dependency
* **Try replicating the failure with same set of tests in the same order**:  If you have your test suite configured to run in a random order, you can set the SEED value to recreate the order.
* **Cross reference each failed occurrence and see if the same tests ran before the failure**: RSpec has a built in bisect tool you can use to help narrow down a set of tests to the ones producing the dependency - however you may find that it runs prohibitively slowly, so sometimes it’s better to look at it manually. Look at each time it failed on CI, cross-reference which test files were run before the failure each time, and see where the overlap is.

#### Preventing order dependency
* **Configure your test suite to run in random order**: otherwise, they’ll only start to appear when you add or remove a certain test. Running in random order is the default in Minitest and is configurable in RSpec.
* **Understand your test setup and teardown process**, and work to close any gaps where shared state to leak through.

### Suspect #3: Time

**Definition**: A test that can pass or fail depending on the time of day when it is run.

#### Example

Imagine we have this code that runs in a `before(:save)` hook on our Task model. It sets an automatic due date to the next day at the end of the day.

``` ruby
def set_default_due_date
   if due_date.nil?
     self.due_date = Date.tomorrow.end_of_day
   end
 end
```

Then we write this test:

``` ruby
it "should set a default due date" do
  task = Task.create
  expected_due_date = (Date.today + 1).end_of_day
  expect(task.due_date).to eq expected_due_date
end
```

Mysteriously, it starts failing every night around 7 p.m.! The trouble is that we’re using two slightly different ways of calculating tomorrow. `Date.tomorrow` uses the time based on the time zone we set for our Rails app, while `Date.today + 1` will be based on the system time. So if the system time is in UTC and our time in zone is EST, they’ll be five hours apart, and after 7pm they’ll be different days, resulting in the failure.

To fix this, you can update your test to use Date.current instead of Date.today, which will respect time zones. 

``` ruby
it "should set a default due date" do
  task = Task.create
  expected_due_date = (Date.current + 1).end_of_day
  expect(task.due_date).to eq expected_due_date
end
```

You can also use the [Timecop](https://github.com/travisjeffery/timecop) gem when you're testing time sensitive logic. This allows you to freeze time to a specific value, so you always know what time your test is running at. 

``` ruby
it "should set a default due date" do
  Timecop.freeze(Time.zone.local(2019, 1, 1, 10, 5, 0)) do
    task = Task.create
    expected_due_date = Time.zone.local(2019, 1, 2, 23, 59, 59)    
    expect(task.due_date).to eq expected_due_date
  end
end
```

#### Identifying time-based flakes

* Are there any references to date or time in the test or the code under test?
* Has every observed failure happened before or after a certain hour of day?
* See if you can reliably replicate failure using Timecop

A strategy you can apply globally to surface time-based flakes is to wraps every test in `Timecop.travel`, mocking the time to a different random time of day on each run, that’s printed out before the test runs. If you're using RSpec, you can accomplish this with a global `around(:each)` block set up in your spec_helper. This will surface failures that would normally only happen after normal work hours during work hours, so you can fix them instead of running into them only when you’re on call and desperately trying to deploy a hot fix at 11pm.

#### Fixing time-based flakes

If the current time could affect the test, freeze it to a specific value with Timecop, or test it with a passed in, static value.

### Suspect #4: Unordered collections

**Definition:** A test that can pass or fail depending on the order of a set of items referenced in it.

#### Example

Here's an example where we’re querying for some posts in the database and then comparing them to some posts that we expect (maybe created earlier in the test):

``` ruby
active_posts = Post.where(state: active)

expect(active_posts).to eq([post1, post2])
```

The issue with this test is that the database query doesn’t have a specific order. So even though things will often be returned from the database in the same order, there’s no guarantee that will always happen. And when it doesn’t, this test fails.

The fix is to specify an order for both sets, so we can be confident we're comparing them correctly:

``` ruby
active_posts = Post.where(state: active).order(:id)
expected_posts = [post1, post2].sort_by(&:id)
expect(active_posts).to eq(expected_posts)
```

#### Identifying unordered collection flakes 

Look for any assertions about: 

* the order of a collection
* the contents of an array
* the first or last item in an array

#### Preventing unordered collection flakes

Fixing this type of flake is relatively simple: use `match_array` (RSpec) when you don’t care about order, or add an explicit sort like in the example.

### Suspect #5: Randomness

**Definition:** A test that can pass or fail depending on the output of a random number generator.

#### Example

So here’s an example of a test data factory that uses FactoryBot to create an event:

``` ruby
factory :event do
  start_date { Date.current + rand(5) }
  end_date { Date.current + rand(10) }
end
```

The issue is that if we have a validation that enforces start date being before end date, tests using this factory will randomly fail when the value produced by rand(10) is lower than the value produced by rand(5). You’re better off just being explicit and creating the same dates every time:

``` ruby
factory :event do
  start_date { Date.current + 5 }
  end_date { Date.current + 10 }
end
```

#### Identifying randomness-based flakes

* Look for use of random number generator - often this is used in factories/fixtures
* Try to see if you can reliably replicate failure with the same `--seed` option added to the command
  * In minitest, this should work out of the box because it's the same as Kernel.srand
  * In RSpec, make sure you’ve set Kernel.srand(config.seed) in spec_helper.rb

#### Preventing randomness-based flakes

* Remove randomness from your tests & instead explicitly test boundaries & edge cases.;
* Avoid using gems like Faker to generate data in tests: they're great for generating realistic dev data, but in your tests it's more important to have reliable behavior & a clear sense of what's being tested than realistically random data.

### Forming a theory & solving it

Now that we've looked at all of the usual suspects, we can move on to forming an theory and actually solving a flaky test mystery.

#### Strategy tips
* **Run through each category & look for identifying signs**: if you have no idea where to start, just go through each suspect one by one and try to make a connection to the test you're looking at.
* **Don’t use trial & error to find a fix - form a strong theory first**: since the feedback loop for testing fixes to a flaky test is typically very slow, it's important not to just guess blindly at solutions - make sure you understand exactly why the fix you're trying could work.
* **Do try to find a way to reliably replicate failures to prove your theory**: once you have an idea of how a flake could be happening, it's ok at that point to use some experimentation to prove to yourself that it's the right fix
* **Consider adding code that will give you more information next time it fails**: if you're at a loss for a theory, trying adding some diagnostic code that will give you more information the next time a test fails.
* **Try pairing with another developer**: flaky tests are great problems to pair with another developer on - it can keep you from going down too many rabbit holes, and forces you to articulate and test your theories together.

#### Is it ever ok to just delete the test?

You have to accept that if you're writing tests, at some point, inevitably, you're going to deal with flaky ones. You can't just delete any test that starts to flake because you'll end up making significant compromises in the coverage you have for you app. And learning to fix and avoid flaky tests is a skill you can develop over time - one that's worth investing in, even if that means spending two days fixing one instead of just deleting it.

That being said, when I'm dealing with flaky tests, I do like to take a step back to think about the test coverage I have for a feature holistically. What situations do I have coverage for, which ones am I neglecting, and what are the stakes of having the kind of bug that might slip through the cracks in my coverage? If the flaky test is for a very small edge case with low stakes, or something that's actually well covered by other tests, maybe it does make sense to delete it.

And this ties into a bigger picture idea, which is that when we're writing tests, we're always making trade-offs between realism and maintainability. Using automated tests instead of manual QA is itself a trade-off in terms of substituting in a machine to do the testing for us, which is going to behave differently than an actual user. But it's worth it in many situations because we can get results faster and consistently and add tests as we code.  Different types of tests go to different lengths to mimic real life, and generally the most realistic ones are the hardest to keep from getting flaky.

![The test pyramid](/assets/images/posts/test_pyramid.png)

There's an idea of the test pyramid - I think first came up with by Mike Cohn, though there's been many other spins on it since - and here's my spin: You should have a strong foundation of lots of unit tests at the bottom. They're simpler, faster, and less likely to be flaky. And as you go from less realistic to more realistic, you should have fewer of those types of test, because they are going to take more effort to maintain, and the tests themselves are coarser-grained - they're covering more situations. End to end tests, like system and feature tests, are just more likely to become flaky since there's so many more moving parts. So it is wise to keep the number of them in our test suite in balance.

#### Fixing flaky tests as a team

* **Make fixing flaky tests high priority since they affect everyone’s velocity**: Since flaky tests can slow everyone down, and erode everyone's trust in your test suite, they should really be high priority to fix - if you can manage it, the next highest priority under production fires. This needs to be something you talk about as a team, communicate to new hires, and agree that it's worth investing time in.
* **Make sure someone is assigned to each active flake & responsibility is spread across the whole team**: I recommend making sure you have a specific person assigned to each active flake. That person is in charge of looking for a fix, temporarily disabling the test in prod while they work on it, if it's frequently flaking, reaching out to others for help if they're stuck, and so on. And it's important to make sure this responsibility is spread out among your team - don't let just one person end up being the flake master and everyone else ignores it. If you're already sending your flakes to a bug tracker as I suggested, you can use that as a place to assign who's working on them.
* **Set a target for your master branch pass rate and track it week over week**: The next thing I'd recommend is setting a target for your master branch pass rate and tracking it week over week - so for example, we want to have builds on our master branch pass 90% of the time. This helps you keep an eye on whether you're progressing towards that goal or not, and course correct if your efforts aren't working.

#### Conclusion

To wrap this all up: if you remember just one thing from this post, I hope it's that flaky tests don't just have to be just an annoying and frustrating problem, or something you try to ignore. Fixing them can be an opportunity to gain a deeper understanding of your tools and your code, and to pretend you're a detective for a little. Hopefully this post has made it easier to do that.