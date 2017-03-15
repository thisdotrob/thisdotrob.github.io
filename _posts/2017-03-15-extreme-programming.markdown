---
layout: post
title:  "eXtreme Programming"
date:   2017-03-15 07:51:00 +0100
categories: XP agile
published: false
---

One of the first agile software development methodologies. Developed primarily by Kent Beck.

Team produces the software in small frequent releases that pass the tests the Customer has defined.

Core principles and practices:
  - Whole team (must include a business representative, "the Customer", may include analyst, manager, coaches. Best teams are generalists with special skills)
  - Planning game:
    1. Release planning (the Customer presents desired features including acceptance tests that will show it is working, programmers give rough estimate of difficulty, the Customer lays out a plan for the project).
    2. Iteration planning (the Customer presents features selected for next 2 weeks iteration, programmers break them down into tasks and perform finer level of estimation).
  - Small releases: at least every iteration, often more frequently (CI). Release reliability and implementation corretness is ensured by by obsession with testing (Customer Tests and TDD).
  - Simple, continuous design (not one-time or up-front)
  - Pair programming ensures all code is reviewed, better tested, better designed, cleaner code and communicates knowledge throughout the team.
  - Test driven development, work in very short cycles of adding a test, making it work and refactoring. All tests must be maintained and run every time a programmer releases code to the repository. Ensures good test coverage, lean code and support as the software design is improved.
  - Refactoring - continuous design improvement. Focuses on removal of duplication and high cohesion/low coupling
  - Continuous integration - catches bugs / problems early, allows shipping features as soon as possible and team gets better at releasing.
  - Collective code ownership, improves code quality and reduces defects as each bit gets many people's attention. Avoids code being put in the wrong place (when programmers can only put it in the place they have responsibility for).
  - Common coding standard, so code looks like it was written by a single competent individual in support of collective ownership.
  - Metaphor, a common vision of how the program the team is working on works
  - Sustainable pace - in it to win, not to die.
