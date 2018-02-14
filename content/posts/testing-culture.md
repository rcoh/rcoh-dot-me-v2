---
title: "To foster a culture of testing, break local deployments"
date: 2018-02-13T23:11:43-08:00
draft: false
prose: true
tags: ["testing", "startups"]
---
Someone recently lamented to me that try as they might, they can't seem to instill a culture of non-manual testing in their team. This problem pervades startups, especially in those with a lot of newer developers. My theory:

> To foster a culture where software engineers are internally motivated to write good tests, **make it harder to run your app locally and easier to write and run tests.** 

To put it another way: rather than investing time in enabling people to run a standalone app on their laptops, invest time in making it easy to write tests. Lead people into the [pit of success](https://blog.codinghorror.com/falling-into-the-pit-of-success/) of effective software testing.

I appreciate that this seems backwards, especially the part about making running a _local_ app worse. 
I observed that since I've started freelancing, I write significantly more tests than I ever did fulltime. My code usually has more test-coverage than the average in the codebase. This isn't because I'm some hardcore TDDer or that I just believe more in the importance of tests than the other people I'm working with -- far from it. The cause, I think, stems not from ideals, but from pragmatics: as a freelance software engineer, I rarely end up doing time-consuming or finicky developer onboarding steps like getting a local version of the app working. **Tests are literally the only way I have to run my code locally.** With no other options, I have no choice but to structure my code in a way I can test it and to write unit and integration tests that come as close as possible to simulating reality. Without meaning to, I lead myself into the pit of success.

If anyone has any stories about how something like this accidental or intentionally changed the way you write tests at your company, I'd love to hear them. If you think this idea is nonsense, I'm happy to hear that too.



