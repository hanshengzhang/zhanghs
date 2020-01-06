---
layout: post
category: tech
title: Lessons I learned on building data pipelines
---

**Background.** I have built data pipelines since I started my career in 2015.
To me, it has been a journey exploring big data knowledges and techniques.
I need to understand the machinism behind resource management and computing frameworks.
I need to learn how analytic databases work and how OLAP queries are executed.
I need to know common data modeling principles and related business processes.
I was very into learning and practicing these techniques. 
However, in 2017, I was asked to give a talk introducing a data mart and there was a "lessons learned" section.
I realized that "*How to write windowing function using HQL*" might not be a good answer. 
Since then, I started to write and adjust my answer to the following question.

**Question.** What lessons have I learned on building data pipelines?

# Lesson 1: Fail as early as possible

For most data pipelines, there are several jobs reading, processing and writing data sets.
It is not easy to find bugs in these jobs because the data sets are usually pretty large, 
Even the bug has been located and fixed, backfilling the 
A data pipeline could fail due to many reasons, such as software bugs, environment limitations, resource shortages and upstream changes.


Development practices.

Don't know how to process. Just fail early. What if passing the wrong values.

Only handle the failure Emit that message. don't catch an exception. 

- use static typed language.
- write unit test.
- write integration test.


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

