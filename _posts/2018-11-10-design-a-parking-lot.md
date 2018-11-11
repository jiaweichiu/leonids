---
layout: post
title:  "Design a parking lot"
categories: [system design]
comments: false
math: true
excerpt: "Typical system design question. Design a parking lot."
---

Wanted to get acquainted with system design interview questions. "Designing a parking lot" seems like a typical question.

Here are some resources:
* [Youtube video](https://www.youtube.com/watch?v=DSGsa0pu8-k) from "Success in Tech"
* [Design an OO parking lot](https://stackoverflow.com/questions/764933/amazon-interview-question-design-an-oo-parking-lot) from StackOverflow.
* [Design an OO parking lot](https://www.geeksforgeeks.org/design-parking-lot-using-object-oriented-principles/) from GeeksforGeeks.

The main skill being tested here I think is not OO, but understanding user requirements. The question seems very vague and we need to ask many questions to understand the requirements and the scope, and learn to take hints from the interviewer.

Some questions to ask:
* What kinds of vehicles do we support?
* What are the sizes of the parking lots?
* Can a bus take up multiple parking lots?
* Is there handicapped parking?
* Do we need to track availability?
* Do we need to optimize the placement? Or do users make API calls to put vehicle at whichever location they like? If so, we need to make sure it makes sense?
* Are there multiple floors?
* Do we need to worry about concurrency?
* Do we need to optimize pricing?

As this is the first post regarding system design interview, there are some other standard questions to ask.
* How many users? How much QPS to support?
* How much data needs to be stored in memory/disk?
* What is the latency requirement?

Ideally, we want to lead the discussion and convince the interviewer that we have a lot of experience building large systems.

There are always tradeoffs. Point them out.

When discussing scalability, you want to recognize the bottlenecks. Maybe use NoSQL instead of relational databases, after some restructuring of your design.
