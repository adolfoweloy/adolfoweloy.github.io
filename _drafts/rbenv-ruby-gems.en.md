---
layout: post
title: "Demystifying rbenv and shims"
date: 2025-05-05 22:02:00 +1100
ref: ruby
comments: true
categories: ruby rbenv gem bundler
description: Taking a peek behind the scenes of rbenv and how shims work
---

In this post I show how rbenv just "knows" which version of Ruby that it should use and how rbenv actually helps a lot with compatibility between Ruby and main ecosystem tools like ruby gems and bundler.

## Motivation

After a long long pause on writing tech stuff, I decided to start writing short posts here again and realised how outdated my Jekyll project was. I have been using an old version of Jekyll and was missing lots of important patches and new features.

It turns out that I was banging my head with Jekyll dependencies, conflicting Ruby Gems and Bundler not working with some versions of Ruby I had installed with rbenv. That was when I decided to understand rbenv shims and more about ruby gems once and for all.

## Context

First of all, why bother using rbenv? Doesn't Ubuntu come with Ruby already? Can't you just use the package managers from your OS? The answer to that is that with these OS package managers I usually don't get a seamless experience switching between programming language versions. I am talking about Ruby here but I also recommend same approach for other languages such as SDKMan for Java, pyenv for Python and so on. With rbenv I can pick any version of Ruby I want and not only that, I can also choose between different runtimes such as JRuby if I want. rbenv also gives me high level of isolation between whatever libraries are installed for each Ruby version.

My direct recommendation? Use a version manager. 
My preference? rbenv

