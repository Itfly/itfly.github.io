--- 
title: Implement an in-memory task queue by BlockingQueue
layout: post
---


In many times we encounter the demand that need to perform lots of same tasks which are produced constantly during our coding work. The tasks need to be consumed rapidly, otherwise they may pile up damaging the system. 

The following examples require this kind of demand:

* A web crawler that constantly browses the web and scans through web pages to create the indices.
* An application that process independent work items in the backgroud and in parallel (such as processing logs asynchronously).

An in-memory task queue can satisfy this kind of demand. A task queue is a data structure maintained by task manager containing queue to store tasks and workers to run.

In this post, we will implement a feasible task queue using Java BlockingQueue. In order to support most of the common demands we need the following features:
* Thread safe
* Multiple workers
* Support retry

Before moving on, let's introduce the Java BlockingQueue first.
