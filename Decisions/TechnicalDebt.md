# Overview

- **Subject**: Whether and how much technical debt we take on and how we pay it back
- **Decision date**: [when was the decision made]
- **Decision contributors**: [list who was involved in the decision]
- **Decision maker**: Engineering Managers
- **Decision owner**: [who executes on the final decision]

# Executive Summary

Octopus will maintain a revolving line of technical credit. We borrow against future earnings into order to move quicker. We will purposefully incur technical debt as well as pay it down. 

# Background

For the first 7 years of it's life, Octopus was a very small company. It managed to produce a great product and do so quickly. Part of this was deliberately taking on technical debt 
and design patterns in order to move faster <sup>[citation needed]</sup>. 

As the company and it's install base has grown the technical debt is starting to slow us down noticeably. It's no longer possible for engineers to learn the whole product and where the sharp edges are. This has cause development to slow down and a reduction in reliability.

# Approach

We accept that we will have an ongoing level of technical debt. The level will shift as the priorities of the product change. The engineering managers are responsible for monitoring this.

Engineers are empowered to incur debt in order to ship features. They should do so deliberately and thoughtfully.

When taking on debt we do not need a plan for how to pay down that specific debt. This will allow us to move faster and often these things are clearer in hindsight.

We will have an indefinite program of paying down technical debt. Currently ~30% of pitch capacity each cycle.

# FAQ

## What is technical debt?

`Technical Debt` is an overloaded term, but at Octopus it encompasses:
- A lean approach to automated testing focusing on the green path in favour of manual testing
- Hairy code that does the job but is hard to understand and modify
- Code and designs that are bug prone
- Monolithic spaghetti design
- The UI, API and DB not agreeing on what a thing is called (ubiquitous language)
- Architecture and designs that have we have outgrown
- Partially completed refactors
- Multiple patterns for doing the same thing

`Technical Debt` is **not**:
- Inconstancies in how features work
- Missing features
- Bad UX
- Bugs (although it may cause bugs)

We use `interest` to describe the consequence arising from the debt, mainly in lost time and customer good will.

## How is technical debt introduced?

Debt can be introduced a number of ways, not limited to:
- Shortcuts to ship a pitch
- Backwards compatibility or versioning requirements
- The trial of a new code pattern
- Lack of knowledge of existing patterns
- The thousand cuts of minor additions and fixes

## Is technical debt bad?

No. As with financial debt, healthy businesses take on liabilities in order to make higher revenue with that the increase in revenue exceeds the repayment on the debt. Businesses also often operate on a revolving line of credit, they never intend to repay the principal.

To apply this to code, a certain level of debt is desired. Paying it all down is counter productive as there is something more valuable that could have been done instead. If Octopus Server ever reaches it's end of life, the debt will evaporate.

## How much technical debt should we have?

> Perfect is the enemy of good - Voltaire

The exact quantity depends on the observer. Not so much that it slows us down significantly and causes many reliability issues. However not so little that we ship no features.

## Is all technical debt equal?

[This article](https://digitalassetmanagementnews.org/opinion/the-technical-debt-metaphor-a-better-alternative/) reframes technical debt as `Call Options`.

It is impossible to predict the future, but we can take an informed guess at it. Code that will likely never change, or change rarely (and is read rarely) would incur less interest than code that is on the main code path and subject to frequent change. For example team management vs the step editor. 

We may never pay off low interest debt.

## Which debt should be pay down?

We should be paying down the highest interest debt first. 

It is often not obvious what is the best way to pay down newly acquired debt. It is also not clear how much interest it will change. It's only after some time that the cost and the way forward becomes clearer.

By targeting well understood debt we can move faster as we won't be making speculative changes on things that may never occur (YAGNI).

# Examples

## API Compatibility and Versioning

As part of a pitch the team decides they need to maintain backwards compatibility. They choose to write code to keep it compatible. For example:
- Re-use an field or term, changing it's meaning without changing it's name (breaking ubiquitous language)
- Modify the existing endpoint to be backwards compatible
- Add a new versioned endpoint

In doing so they introduce additional code paths or inconsistencies that require maintenance and may confuse future developers.

They add metrics to measure the usage of the old code paths.

This debt is taken on with the understanding that we will consider whether and how to remove it at a future date.

This allows us to meet a customer requirement without planning too far into the future. 

## Architectural Trials

After consultation, an individual or team introduces a new architecture, pattern or design. 

The limit it to one section of the application. This may be because they are not sure the pattern is useful, or because there is not enough time.

The code is left with two ways of doings things.

Whether to remove it, gradually spread it, or undertake a pitch to complete it is left for the future.

## Outgrown

As the product grows and the userbase changes, previous assets may suddenly become liabilities. The incurred debt could not have reasonably forseen, or if it was it was very distant and nebulous. 

This is unavoidable.

## Experiments

The team is tasked with writing an experimental feature. The goal is to find the rabbit holes and field test the idea. 

The team deliberately "hacks it in" and focuses on the green path in order to get more features out the door.

This debt is taken on with the understanding that quick iteration and feedback is important for us and a competitive advantage.

We usually commit to either finish it properly, or removing it.

## The Spaghetti

The deployment creation code has had many small changes and bug fixes by many hands. No-one had the time, inclination or skill to rework it.

We will identify these when they become a problem and invest in fixing them.