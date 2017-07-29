---
layout: post
title:  "How to Analyze A Source Code Repository"
date:   2017-07-18 19:59:03 -0800
---
After writing thirty articles regarding Github source code reading, I want to summarize a template for approaching and understanding the codes. This effort can help quickly understand the key pieces in a large repository, and make the future blog writing faster. Also, the techniques described here can be used in other scenarios such as reviewing code and debugging.

Ideally, a comprehensive code reading should answer the questions listed in each subsections.

## Background and Overview
Key questions are:

1. Where did you find this repository?
2. What does this code repository do? Can you provide some concrete examples?
3. Why is this repository worth discussing?
4. For blog writing, if this is a big repository, can we divide this repository into two posts? If yes, discuss the focus in the first blog.

## The Repository Structure
Key questions are:

1. At a high level, what are the major components of the code?
2. Can you draw a diagram to visualize the different components?
3. What tool or IDE can we use to better understand the code?
4. What can we learn from the structure?

A [good article](https://github.com/aredridel/how-to-read-code/blob/master/how-to-read-code.md) can be found online. To summarize, it divides the source codes into six types based on functionality. It also provides a set of questions to reason when reading each type.

1. Glue: error handling? convert what to what?
2. Interface-defining: error handling? complete interface?
3. Implementation: input validation? tests? what might break? difficulty adding new features?
4. Algorithmic: special case for implementation.
5. Configuring: environment variables? security sensitive info?
6. Tasking: Transactionality? Resumabilty?

## Get Your Hands Dirty
Now we are at a stage to dig deeper and figure out how the key functionalities are coded. We usually walk through a couple of key code paths, and illustrate how they work with flow charts. Some key questions are:

1. Can you build and run the code?
2. If possible, can you summarize this code repository with merely 100 lines of code?
3. Read and run unit tests?
4. Do you understand the invariants and hidden assumptions?

## Tricks, New inspirations, and interesting code snippets to learn from
Questions are:

1. Did you learn something new from reading the code repository? It can be best practices, debugging tricks, testing tricks, code crunching skills, design patterns and Good coding habit.
2. It is worth pasting some good code snippets for future reference.

## Key Takeaways And Discussions
When talking about takeaways, we will summarize the learnings, and optionally compare it with other repositories.

1. Discuss how it works, and useful things you learned.
2. Are there other similar repositories that implement similar functionalities? What are the differences?
3. What this repositories does well, and not so well?
4. Can there be a different way of doing things?
