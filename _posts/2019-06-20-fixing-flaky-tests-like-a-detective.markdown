---
layout: post
title:  "Fixing Flaky Tests Like a Detective (RailsConf 2019)"
date:   2016-04-13 18:42:49 -0500
tags: ["Ruby", "Ruby on Rails", "Testing"]
---

This spring I spoke at RailsConf about fixing flaky tests, and how reading lots of mystery novels helped me learn how to do it. Here's the video of my talk - if you prefer to read about it instead, I've also adapted it into a blog post below.

<iframe width="560" height="315" src="https://www.youtube.com/embed/qTyoMg_rmrQ" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## My first encounter with a flaky test

I want to start out by telling you a story. It was my first year as a software engineer, and I’d worked really hard building out a complicated form - my first big frontend feature. I wrote lots of unit and feature tests to make sure I didn’t miss any edge cases. Everything was working great, and we shipped it! But then, a few days later, it started.

A red build on our master branch. The failing test was one of the feature tests for my form. Huh?? It had been working fine for days!

The first time it came up, we all kind of ignored it. Tests failed randomly once in a while.

But then it happened again. And again.

I thought, no problem, I’ll spend an afternoon digging into it and fix it.

The only problem was: I had never fixed a flaky test before and had no concept of why a test would pass or fail on different runs. 

So I did what I often did when debugging problems I didn’t really understand at all… I tried to use trial and error: making a guess at a random change without really understanding how it would help.

That kind of approach sometimes worked with normal bugs - you could start by using trial and error, quickly ruling out various potential solutions, and eventually you'd hit on the right one, at which point you could reason backwards about why it worked. But that didn't help at all with this flaky test, because I could never be sure if my random fixes worked. I'd run it with the fix 50 times without seeing a failure and think I solved it, but the next day we'd see it fail agian. 

And that’s exactly what makes fixing flaky tests so challenging. Trying random fixes and testing them by running the test over and over to see if it fails doesn’t work well and is a very slow feedback loop.

We eventually figured out a fix for that flaky test, but not until several different people had tried random fixes that failed and it had sucked up entire days of work.

The other thing I learned around that time is that even just a few flaky tests can really slow down your team. When a test fails without actually signaling something wrong with the test suite, you not only have to re-run all the tests before you’re ready to deploy your code, which slows down the deployment process,

You also lose some trust in your test suite, and eventually might even start ignoring real failures because you assume they’re just flakes.

So it’s super important to learn how to fix flaky tests efficiently, and better yet avoid writing them in the first place.

## Discovering a method

For me, the real break through in figuring out how to fix flaky tests was when I came up with a method. 

Instead of starting with throwing fixes at the wall and seeing what stuck, I **gathered all the information I could** about the flaky test and the times that it had failed. Then, I tried to **fit it into one of the 5 main categories of flaky test** - we’ll talk about what those are in a minute - and based on that, I **came up with a theory** of how it might be happening. Then, I implemented my fix.

At the same time that I was figuring this out, I was on kind of mystery novel binge. And it struck me that every time I was working on fixing a flaky test, I felt kind of like a detective solving a mystery. After all, the steps to do that (at least in the novels I read, which are probably very different than real life) are basically:

You start with **gathering evidence**, then you **identify suspects**, **come up with a theory of means and motive**, and then you solve it. Thinking about it that way made fixing flaky tests more enjoyable and it actually became a fun challenge for me instead of a frustrating and tedious problem.

So that's the framework I'm going to use in this talk for explaining how to fix flaky tests. Let's start with step one: gathering evidence.

## Gathering evidence

There are lots of pieces of information that can be helpful to have when you’re trying to diagnose and fix a flaky test. They include:

* Error messages and output from each failure
* Time of day failures occurred
* How often the test is failing
* Which tests were run before, in what order

So how can you efficiently get all this information? 

A method that I’ve used in the past is to have any time a test fails on your master branch - or whatever branch you would not expect to see failures on since tests had to pass before merging any changes - have any failures on that branch automatically sent to a bug tracker with all the metadata you’d need including a link to the build where they failed. I’ve had success doing this with Rollbar in the past, but I’m sure other bug trackers would work as well. 

When doing that, it’s important to make sure the failures for the same test can generally be grouped together - it might take a little configuration or finessing to get this to work, but it’s really helpful so that you can cross reference between different failures and figure out what they might have in common.

## Narrowing down your suspects

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

#### Async flake examples

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

#### Diagnosing async flakes

* **Is it a system or feature test** (AKA, some kind of test that interacts with your app via the browser?). If it is, look for any events in it where you aren't waiting for the result before moving on.
* **Capture some screenshots**: With most Capybara drivers, you're able to call `save_screenshot` at any point to save a screenshot of the page, usually both as html and a png. You can also use the [capybara-screenshot](https://github.com/mattheworiordan/capybara-screenshot) gem, which helps you ensure that any time a test fails, a screenshot is saved. 

#### Preventing async flakes

* **Make sure your test is waiting for each action to finish**: break your test down into individual actions, and check that in between each of them, you're waiting for a clear sign that the results of that action are complete.
* **Don’t use `sleep` - wait for something specific**: using `sleep` is unreliable & slower than waiting for a specific event
* **Understand which Capybara methods wait & retry, and which don’t**: Make sure you’re familiar with Capybara’s API and know which of its methods will wait. Everything based off “find” will wait, but certain things, like “all”, will not wait.
* **Check that each assertion is working as expected**: the alphabetical order was a great example of a situation where it looked like we were doing everything right, but because things weren't happening under the hood the way we thought they were, we ended up with a flake.

### Suspect #2: Order dependency

Order dependent tests are those that can pass or fail depending on which tests ran before them. Usually, this is caused by some sort of state leaking between tests - so when the state another test creates is present (or not present) it can cause the flaky test to fail.

Order dependency can sneak into your test suite in several different ways, including: 
  * database state 
  * global and class variables in your app
  * browser state, for system & feature tests that use a browser.


#### Order dependency caused by database state

When you're writing tests, each test should start with a "clean" DB - that might not mean with a fully empty database, but if anything is created, updated or deleted in the database DURING a single test, it should be put back the way it was at the beginning. This is important because otherwise, those changes to the database could have unexpected impacts on later tests, or create dependencies between tests so you can't remove or re-order tests without risking cascading failures.

Database cleaning in Rails should "just work" and often does, especially if you're able to use Rails' basic, built in transactional cleaning. But there are so many different ways you might have your Rails test suite configured, and it _is_ possible to do it in such a way that certain gotchas are introduced. So it's important to know how your database cleaner works, when it runs, and if there's anything it's leaving behind - especially if you're dealing with a flaky test that seems to be order dependent.

There are several different ways to handle clearing your database state: 

##### Wrapping your test in a transaction and rolling it back after the test

This is generally the fastest way to clear your database, and it's  the default for Rails tests, but in the past you couldn’t use it with Capybara tests because the test code and test server didn’t share a database connection. 

Rails 5 system tests addressed that issue by allowing the test code and test server to have shared access to a database connection during the test, so they are both looking at the same transaction

This approach still requires you to be careful about a couple of things. Transactions might have slightly different behavior than you see in your actual app - for example, if you have any after_commit hooks set up on your models, they won’t be run.

##### Using truncation of tables or deletion of records touched during the test

Because of the limitations of transactional cleanup, many Rails test suites use the [database_cleaner](https://github.com/DatabaseCleaner/database_cleaner) gem to clean with either truncation - truncating all tables touched in the test - or deletion, which uses DELETE FROM to delete all records created in a test. 

Both deletion and truncation are generally be slower than transactional, but since they let you run the app just like it normally would — without extra transactions wrapping each test — they’re a little more realistic and that can make them easier to work with. 

If you use database_cleaner, you should also make sure you’re running it in an append_after block, not an after each block- this ensures it runs after Capybara’s cleanup, which includes waiting for any lingering ajax requests that could affect the state of the database and need to be cleaned up.

### Order dependency examples

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

### Other sources of order dependency
* **The browser**:  since tests run within the same browser, that can contain specific state depending on the test that just run. Capybara works hard to clean this all up before it moves on to the next test, so this usually should be taken care of for you, but it's likely possible that depending on your configuration you could have some issues, so it's good to be aware of that as a possibility.
* **Global/class variables**:  if you modify those, that can persist from one test to the next. Normally Ruby will yell at you if you reassign these, but one area where they can sneak through is if you have a hash and you just change one of the values in it - since that isn't reassigning the entire variable, it won't come up as a warning.

### Identifying order dependency
* **Try replicating the failure with same set of tests in the same order**:  If you have your test suite configured to run in a random order, you can set the SEED value to recreate the order.
* **Cross reference each failed occurrence and see if the same tests ran before the failure**: RSpec has a built in bisect tool you can use to help narrow down a set of tests to the ones producing the dependency - however you may find that it runs prohibitively slowly, so sometimes it’s better to look at it manually. Look at each time it failed on CI, cross-reference which test files were run before the failure each time, and see where the overlap is.

### Preventing order dependency
* **Configure your test suite to run in random order**: otherwise, they’ll only start to appear when you add or remove a certain test. Running in random order is the default in Minitest and is configurable in RSpec
* **Understand your test setup and teardown process**, and work to close any gaps where shared state to leak through