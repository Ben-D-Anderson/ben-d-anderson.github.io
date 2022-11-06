---
layout: post
title: Differences Between Intermediate And Enterprise Code
description: Here I discuss the design and style differences between an individual's intermediate code and a skilled team's enterprise code. I'll also suggest strategies that can be used to ensure developed projects are high quality.
readtime: 7 minute
tags: programming
---

There are many differences between the code an individual hobbyist will write when experimenting with a spontaneous idea, and a meticulously designed solution that is implemented by an enterprise team. Said differences can even be recognised by paying particular attention to the terminology I used in the previous sentence.

The differences result from following a few key ideas:
- [Planning](#project-planning)
- [Using the right tools for the job](#using-the-right-tools-for-the-job)
- [Writing clean & readable code](#writing-clean--readable-code)
- [Future-proofing](#future-proofing)

Believe it or not, these concepts are all it takes to create industry-leading software. The catch is that they can be implemented to varying degrees, for example: you may only 'design' your system to the extent that you modulise the solution; however, this doesn't take into account how said modules independently function or link together to provide the most suitable, efficient solution.

## Project Planning
Project planning is by far one of the most significant stages of any successful project, mainly due to the fact that it can completely change how the problem is approached and how the solution is structured.

The main elements of project planning are as follows:
- Problem analysis
- Problem decomposition
- System design
- Work delegation

The last one is specific to teams, but there's nothing stopping an individual from planning how they will use their time to be as effective as possible.

When working on my own projects, I find the first two bullet points to actually be critical stages. If you ever find yourself programming, and at the same time, changing your mind about what features you want to include; you have either failed to properly analyse the problem you are trying to solve, or failed to distinguish which features are actually required for your minimum viable product (MVP).

One way you can effectively determine which features or systems you need to implement in your project, is to evaluate other similar projects - even if your problem is unique, there will still be elements of it which other engineers/researchers have solved before. This can make it much easier to plan out a system, as another team has put lots of time into meticulously designing an intricate solution which you can use as a template.

If parts of your probject have never been developed before, then you're lucky enough to be able to do some original [system design](https://www.geeksforgeeks.org/system-design-tutorial/). There's no secret to designing the perfect system, it takes a lot of knowledge in a variety of fields and ultimately boils down to having an in-depth understanding of each system you are attempting to incorporate. Luckily, if you are just an individual working on a project, the system you design very rarely needs to be optimised, super-efficient or ultra-scalable - so your design choices likely aren't going to have a catastrophic impact. But if you're not working alone on a small project, then I'll touch more on why systems design may actually be crucial to your project, in the section on [future proofing](#future-proofing).

## Using The Right Tools For The Job
Far too often, developers re-invent the wheel and write code that already exists - this wastes time and often produces a worse solution than if they had used existing, established tools.

As soon as you have finished planning your project, you should begin researching the following:
- Frameworks
- APIs
- Libraries
- Programming languages (considering their ecosystem)
- DevOps tools

If your project is going to be built on top of a framework, ensure that every pro and con of your chosen framework is thoroughly considered - a framework can make or break a project. Choosing the right framework can result in minimal code being written and a relaxing developer experience. Furthermore, migrating to a different framework at a later date can be an incredibly painful process.

Whilst APIs introduce a dependency on third party services, they can often reduce the complexity of a project. You should carefully analyse the reliability requirements of your solution (ideally during the planning stage) in order to determine whether external APIs are right for you. If you decide to use an API, consider possible financial implications of doing so, and be sure to check [this list of public APIs](https://github.com/public-apis/public-apis).

Established and reputable libraries can save you from writing lots of code, a good library may even sway you to choose a particular programming language. However, be cautious of libraries which are rarely maintained and were only developed by one person, discovering a bug in their code could force you to change the library you use.

I cannot emphasise enough that **not every project will require the same tools**. If you use the same programming language for every endeavour, your projects will suffer. Different languages have different use cases. For example, if you tried to write a desktop application in PHP, it would not be easy; it's not impossible, but I wouldn't recommend it.

DevOps tools can supercharge your development workflow. Companies like Netflix have CI/CD systems in place that allow them to deploy thousands of times per day. But if you don't require an efficient deployment mechanism, or you're just running solo on a project, then it might not be worth going 'DevOps crazy'. Just be mindful that there will definitely be some tools out there that can improve your developer experience whilst consuming minimal time.

## Writing Clean & Readable Code
One of the most natural ways to describe clean code, is what Linus Torvalds calls ['good taste'](https://youtu.be/o8NPllzkFhE?t=863). There are probably hundreds, if not thousands, of pieces of literature that attempt to explain how to write good code, so I'll leave that as an excercise to the reader - but just be aware that it takes a strong understanding of data structures, algorithms and computer science in general, as well as lots of experience actually writing code.

The programming language you are using will have its own unique features and ways of doing things, therefore, in order to write the best code possible, it is very useful to have a depth of knowledge and experience in said language. However, you should be careful about trying to write the 'best code possible', because all-too-often developers rewrite the solution in different ways, desperately searching for the most elegant code - you will never be 100% satisfied with your code.

Code must be expressive, in the sense that other developers should be able to read and understand your code (assuming they are familiar with the language) without needing to read excessive documentation or comments. In cases where this is not possible, such as for a complex algorithm, code should be commented to the extent that the comments add information which couldn't previously be derived or infered from the code itself.

## Future Proofing
Future proofing is often described as the difference between a 'software developer' and a 'software engineer'. Instead of 'developing' a system, you are 'engineering' it to be durable and effective.

Scalability is an absolute must when it comes to future-proof software, there are countless essays on designing scalable software so I'd recommend doing some further reading if you would like to know more. However, my main piece of advice for making software scalable is to decouple components. That way, small components can change and grow to meet demand, but the overall system remains mainly unaffected.

Data should be handled very carefully and log data should be stored judiciously. I would comment further on databases and how integral data is to software, but nothing I say will come close to the masterpiece that is the book ["Designing Data Intensive Applications"](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781491903063/). I'd consider it essential reading for anyone dealing with, or planning to deal with, large volumes of data.

My final point may not necessary be considered future proofing, but maintaining uptime and reliability of services is essential if anyone uses your software.
