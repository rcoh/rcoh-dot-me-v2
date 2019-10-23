---
title: "How to Code Review"
date: 2019-10-23T09:59:22-07:00
draft: false
tags: ["eng-leadership"]
---
_This document is adapted from an internal document I wrote for one of my clients, [LogicHub](https://www.logichub.com/). They've graciously allowed me to adapt it and share it as the blog post you're reading now._

As a reviewer, your primary goal is to find bugs before code is merged. Your secondary goal is to keep the code quality as high as possible, within reason and the time constraints of the moment. Depending on the situation, code reviews can also be a valuable venue for mentorship.


## Key Components
The guide that follows is quite prescriptive, but there are a few components I think are generally crucial to giving a good review on a non-trivial piece of code:

1. Taking multiple passes: Providing feedback that goes beyond line-by-line nits is really only possible when you are viewing code a second time (or more) with the added context of how it fits into the bigger picture.
2. Don't just skim the tests: Tests are runnable documentation of the code's behavior. Rather than skimming them last as an afterthought, reading them early provides key context for the rest of the diff.

This guide won't make code reviews easy and it doesn't magically find bugs. But it's a big step up from skimming through a piece of code once, and expecting to find subtle issues.

## How to Code Review: A step-by-step guide

0. If you have a wide monitor, open this guide next to the diff you're looking at. Following along helps ensure you don't forget steps.
1. Get some context on what's being changed and why. Reconcile what the author says they're changing with a relevant requirements doc (JIRA story, design doc, etc.). Evaluate whether all the requirements are met.
2. It's time to dig into the code. The tests are a great place to start -- it's easy to skim through the tests and assume they're correct. But: the tests are your opportunity to find out what parts of the code have been proved correct and which haven't. The tests are your first hint about bugs that might be lurking. Tip: Tests usually end up at the bottom of the diff.

3. After you've reviewed the tests, I recommend taking a quick skim just to get the lay of the land and understand which files are changed and how. Look for typos in comments and variable names, but ignore everything more substantive -- don't get distracted! Your goal is to gain a high-level summary of what the author changed and why.

4. After you get the lay of the land, I like to do a review for style and code clarity. If your company has a style/coding conventions guide, this is a good doc to have open while doing this part of the review. Try to make sure that the diff follows existing best practices. Look for local bugs you can find without needing to think about the diff in its entirety. For example:
    - Improperly handled error conditions / exceptions
    - Obvious race conditions
    - Language feature misunderstandings, eg. in Python:

        ```python
        try:
          ...
        except IOError, ValueError: # doesn't catch both, it aliases IOError to ValueError
          ...
        ```
    - Unchecked invariants eg. `array[0]` when the array might be empty.

    It's nice to have a company best practices guide here that you can go through. This is especially important if your code base contains a lot of code that _doesn't_ follow current best practices since it's easy to copy/paste a pattern without realizing that it has major shortcomings.

    Sometimes this section of the review can be nice to do backwards to help really focus on line-by-line details without your brain skimming over code you think is correct.

    It might seem like a waste to focus on the nitty-gritty details when you might ask the author to refactor the entire diff later -- I don't disagree. Taking a pass to focus on the small is _really_ an excuse to read closely through the diff multiple times. This way, by the time you get to looking at the real substance of a diff you really understand what's going on. 

5. Your next job is to think about possible bugs that are not covered by the tests. It can sometimes be a nice (but time consuming) exercise to check out the branch locally and write these tests to determine if your intuition is correct.

6. In the previous phase, you looked at the code in a narrow scope, thinking about individual line-by-line bugs. Now, it's time to dig into the meat of the review. As a stretch goal, strive to understand the code with the same amount of understanding you'd have if you had written the diff yourself. Focus on areas of complexity and mutable state. Think about what has test coverage and what does not. If you have time, check out the branch and make some tweaks. Do the changes have the expected impact? If you intentionally introduce a bug, does it produce a failing test? Think about the diff as a whole. Here is an incomplete list of some other questions to get you started:
     * Is it backwards compatible? 
     * Engineers will frequently test on a local deployment with little or no data. How will the change behave if data already exists? 
     * How will this change impact overall system performance? 
     * Can it be done simpler? Is it engineered for requirements that don't exist? Does it fail to satisfy key requirements?
     * Will this code work tomorrow? Will it work next year? What implicit assumptions does the author make? Did they document them in comments?
     * Is this code written to be testable?
     * How will this code interact with other features?
     * If the diff creates database tables are they appropriately indexed?
     * If you're asked to fix a bug in this diff tomorrow, could you do it? Or is it too complex to understand?
     * Was some of this code copied and pasted from elsewhere? Look very carefully to ensure everything was changed appropriately.

7. Give clear, direct comments. The tone you strike in your review feedback is a personal decision, however, there is one rule I think is crucial to a healthy culture of code review: Do _not_ write comments in a way that criticize the author. You are reviewing the code, not the code's author. If the author feels personally attacked, they won't be receptive to your feedback. 

    * **Bad:** What were you thinking? Did you even test this? 
    * **Good:** I'm having a hard time following this code and I'm not sure it works. What manual testing was performed?

    If the same issue exists in many places, you only need to comment on it once, but note that it exists in other places.

    If the author does something great, either in the original diff or in response to a comment, let them know! Code reviews don't need to only address shortcomings or issues!

After making comments, it's time to make a final determination. Your company should decide on a social contract of what the different responses on GitHub (or your platform of choice) mean. Here's what the three options GitHub offers mean at LogicHub:

1. **Comment**: I'm just passing through this diff and had some things to add. I don't have enough context to decide if this can be accepted yet, but I also didn't see show-stopper problems either. _This is only an acceptable response if you are not `Assigned` to the diff. If you have the final say, take a stand one way or the other._
2. **Accept**: I trust the author to implement the changes I've requested (or I have no suggestions). They should land the diff without further input from me. If the author has questions, it is their obligation to follow up with me.
3. **Request Changes**: This code has problems. I want to look at the diff again before it is landed. I will click "Resolve" on my comments to indicate that I'm satisfied with the fix.

## Closing Thoughts
Although following a set of steps won't guarantee that you do a good code review, I've found that it's a great way to force yourself to gain a deeper understanding of the code at hand. These steps won't be right for every diff, but hopefully they provide a framework to start from. To use this guide most effectively at your own company, each step should be augmented with things to look for specific to your architecture and tech stack.

***
{{% subscribe %}}