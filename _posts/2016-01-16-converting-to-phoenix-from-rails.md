---
layout: post
title: Converting to Phoenix from Rails
---

I have been doing full time dev in Rails for almost 10 years now and the itch to give something else a serious play has been getting stronger and stronger of late.  As a backend focused developer I seem to constantly battle heavy loads, trying to push them off the request and into the background, so the parallelism and distributed aspects of Elixir appeal to me. Not to mention it’s speed compared to Ruby.

Over the last couple of years I’ve developed a sizeable IoT setup at home with over 100 sensors involving RaspberryPI, BeagleBone Black and Mac hardware collecting and processing all of the data flooding in. Running on this is a mixture of Rails and NodeJS with [RabbitMQ](https://www.rabbitmq.com/) at the core. Over the last few months I’ve stopped development of it as I plan to switch over to Elixir and Phoenix as that seems like a much better fit for where I want to take my system - The Homeatron9000.

So far I’ve read most of the elixir-lang.org [Getting Started](http://elixir-lang.org/getting-started/introduction.html) section, the first ~100 pages of [Programming Elixir](https://pragprog.com/book/elixir12/programming-elixir-1-2), and have recently reached the Testing chapter in [Programming Phoenix](https://pragprog.com/book/phoenix/programming-phoenix). It’s only now that I’m starting to try my own hand at some code, first up I’m jumping straight into replacing my main Rails app. As I do this I’ll be talking here about the move from a Rails developers point of view, the good, the bad, the ugly and hopefully the pretty.
