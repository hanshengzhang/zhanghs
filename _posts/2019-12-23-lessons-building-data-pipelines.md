---
layout: post
category: tech
title: Lessons I learned on building data pipelines
---

**Background.** 
I have built data pipelines since I started my career in 2015.
To me, it has been a journey exploring big data knowledges and techniques.
I need to understand the machinism behind resource management and computing frameworks.
I need to learn how analytic databases work and how OLAP queries are executed.
I need to be familiar with common data modeling principles and the ads/marketing business processes.
I was very into learning and practicing these techniques. 
However, in 2017, I was asked to give a talk introducing a data mart and there was a "lessons learned" section.
I realized that "*How to write windowing function using HQL*" might not be a good answer. 
Since then, I started to maintain a list for the following question.

**Question.** 
What lessons have I learned on building data pipelines?

**Methodology.** 
I adjust the list from time to time based on my experience of building and maintaining data pipelines.
Locating and fixing pipeline failures accounts for the major part of it. 
Thus, most of (if not all) these lessions are about avoiding pipeline failures.
I've also helped several junior coworkers to build or debug their data pipelines.
It helps a lot for me to know what lessons are useful and in which cases they are.

# Lesson 1: Fail early

A data pipeline could fail due to many reasons, such as software bugs, environment limitations, resource shortages and upstream changes.
It could be very time-consuming to diagnose a data pipeline failure.
Even when the bug has been located and fixed, sometimes you need to perform a data backfilling.
My suggest is that if a pipeline has to fail, fail as early as possible.

**Try to fail during the development.**
Evaluate whether a static typed language should be used to avoid incorrect type definitions and castings.
Write unit tests for complex data processing logics.
Write integration tests before deploying the pipeline. 

**Validate the input and output.**

**Catch only the expected exceptions.**

**Note.** There is an article named [*Fail Fast*](https://www.martinfowler.com/ieeeSoftware/failFast.pdf). 
I didn't notice it until recently. You might find some ideas are shared.




# Lesson 2: Magic is trouble (2017)

- write scala codes when you know how they are compiled to Java.
A bound it is possible. Beyond that bound.

# Lesson 3: Design the states (2018)

Essential. idempotence.

# Lesson 4: Validate the assumptions (2018)



# Lesson 5: Make the component act correctly (2019)

# Lesson 6: Implement two-stage compatibility (2019)

