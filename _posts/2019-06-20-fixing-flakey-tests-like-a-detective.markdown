---
layout: post
title:  "Fixing Flakey Tests Like a Detective - RailsConf 2019"
date:   2016-04-13 18:42:49 -0500
tags: ["Ruby", "Ruby on Rails", "Testing"]
---

## My first encounter with a flaky test

I want to start out by telling you a story. It was my first year as a software engineer, and I’d worked really hard building out a complicated form - my first big frontend feature. I wrote lots of unit and feature tests to make sure I didn’t miss any edge cases. Everything was working great, and we shipped it! But then, a few days later, it started.

A red build on our master branch. The failing test was one of the feature tests for my form. Huh?? It had been working fine for days!

The first time it came up, we all kind of ignored it. Tests failed randomly once in a while.

But then it happened again. And again.

I thought, no problem, I’ll spend an afternoon digging into it and fix it.

The only problem was: I had never fixed a flaky test before and had no concept of why a test would pass or fail on different runs. 

So I did what I often did when debugging problems I didn’t really understand at all… I tried to use trial and error. I made a random change, and then ran the test over and over again to see if it could still fail. That kind of approach sometimes worked with normal bugs - you could start by using trial and error and sometimes that would lead you to a solution that also helped you better understand the actual problem you were dealing with. But that didn't help at all with this flaky test. I tried a random fix, ran it 50 more times without it failing, but then a few days later it still failed again.

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
