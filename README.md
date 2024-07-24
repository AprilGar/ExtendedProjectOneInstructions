# Building a RESTful API with Scala and Play

*Based around a tutorial from [here](https://spr.com/building-a-simple-rest-api-with-scala-play-part-1/) and [here](https://spr.com/building-a-simple-rest-api-with-scala-play-part-2/).*

## This Tutorial
This tutorial walks you through the creation of an API that can store, retrieve and update data in a Mongo database. It covers these main concepts and technologies, some more in depth than others:
* Scala
* Creating a RESTful API with Play Framework
* MongoDB access using a Reactive Mongo driver
* Test-driven development
* git and GitHub
* SBT
* Mockito/MockFactory
* CURL 
* HTTP request methods and response statuses (brief guide [here](https://code.tutsplus.com/tutorials/http-the-protocol-every-web-developer-must-know-part-1--net-31177))

Some of the sections are purposefully vague and get more challenging as you progress. This is intended - your peers, your Teams Academy channel, Stack Overflow, Google, GitHub, your notes and your trainer are all valuable resources! 

Remember to utilise the 4B's methodology when something isn't going quite to plan: brain, book, buddy, boss. 

## Why Scala & Play?
Reactive Programming is based around building applications that can meet the diverse demands of modern environments. Four main characteristics:
* Responsiveness: high-performance, with consistent and fast response times
* Resilience: fails gracefully, and remains responsive during failures
* Elasticity: remains responsive during varying workload
* Message Driven: non-blocking, back-pressure capable, asynchronous

## Play Framework
[Play! Framework](https://playframework.com) is an open source web app framework. It’s similar to other web application frameworks you may be familiar with like Spring MVC and Rails:
* model, view, controller-based
* comes with a lot of built-in support tooling (scaffolding, execution, dependency management, etc)
* based on the principle of convention over configuration

In other ways, it differs from those frameworks: it’s 100% stateless and is built to run on [Netty](http://netty.io) – a non-blocking, asynchronous application framework.

## Reactive Mongo
Behind Play, [Reactive Mongo](http://reactivemongo.org) will give us non-blocking and asynchronous access to a Mongo document store through a Scala driver and a Play module for easy integration into a Play apps.

The Reactive Mongo API exposes most normal data access functions you’d come to expect, but returns results as Scala Futures, and provides translation utilities for translating the Mongo document format (BSON) to JSON, and many functional helper methods for dealing with result sets.

For this tutorial, we'll be using [the HMRC mongo](https://github.com/hmrc/hmrc-mongo) library.

## SBT
[SBT](https://www.scala-sbt.org/0.13/docs/Getting-Started.html) (Simple Build Tool) supports the compilation of Scala code and integration with other frameworks we're using. It is similar to Maven which you may be familiar with. 

## Unit Testing
To wrap it all up, we’ll be using Scala Test and MockFactory to unit test our application. Scala Test allows us to write test cases in the style of behaviour-driven development and highlights the flexibility of Scala and how it an easily be used to create a domain-specific language.

## Sections
* [Part 1](Building-A-RESTful-API-With-Scala-Play/Part1.md) 
* [Part 2](Building-A-RESTful-API-With-Scala-Play/Part2.md) 
* [Part 3](Building-A-RESTful-API-With-Scala-Play/Part3.md) 
* [Part 4](Building-A-RESTful-API-With-Scala-Play/Part4.md)
* [Part 5](Building-A-RESTful-API-With-Scala-Play/Part5.md)
