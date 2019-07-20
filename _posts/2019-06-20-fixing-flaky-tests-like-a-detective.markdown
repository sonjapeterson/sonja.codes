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

### Async code

Async code is, in my experience, one of the biggest sources of flaky tests, especially when testing Rails apps.

When I say “async code”, I’m referring to tests in which some code runs asynchronously, which means that events in the test can happen in more than one order

The most common way this comes up when testing Rails apps is in your system and feature tests. Most Rails apps use Capybara, either through Rails’ built in system tests, or RSpec feature tests, to write end to end tests for their application that spin up a Rails server and a browser, and interact with the app like a user would.

The reason you’re necessarily dealing with async code and concurrency when you write Capybara tests is that there at least 3 different threads involved: 

* The main thread executing your test code
* Another thread spun off to run the server for your Rails app
* A thread in a separate process running the browser, which Capybara interacts with using a driver

Let’s look at how this plays out in a simple example. Imagine you have a Capybara test that clicks on a Submit Post button, and then checks that a post was created in the database.

``` ruby
click_on "Submit Post"

expect(Post.count).to eq 1
```
