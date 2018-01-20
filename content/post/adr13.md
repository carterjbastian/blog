+++
title = "Re-architecting for Dependency Structure"
description = "Untangling Khan Academy's Webapp for Fun and Profit"
date = 2018-01-15
publishDate = 2018-01-15
author = "Carter J. Bastian"
aliases = [
    "/post/adr-13/",
    "/adr-13/",
    "/re-architecting-for-dependencies/",
]
+++

\[ This article is cross-posted from Khan Academy's engineering blog. You can see the original [here](http://engineering.khanacademy.org) \]

For the last two months of 2017, the Khan Academy engineering team has been working on our Technical Sustainability Milestone---an all-hands-on-deck initiative aimed at reducing tech debt and optimizing for effective development in 2018 and beyond. One problem that we decided to tackle during this milestone was python dependency management. 

Before the project described here, the backend of our webapp was a giant, unstructured tangle of dependencies. The lack of an intentionally-architected dependency structure made it difficult both to build new features and to maintain our current product. 

In this post, I'm going to take at our attempt solving for dependency management, as well as the challenges that we ran into over the course of the project.

## Background

Before discussing our solution, it's worthwhile to get a clear picture of the problem. Python doesn't enforced many rules on the structure of your codebase. As a result, it's syntactically correct for almost anything module to import another python module. However, freedom quickly becomes dangerous. 

There are a lot of potential issues that can arise from bad dependency structure. The most immediate and noticeable problem is circular dependency, where two or more different pieces of code mutually depend on each other. For example, we could have the following scenario, where `module_a` depends on `module_b`, and `module_b` depends on `module_a`:
```
# The -> relation indicated dependency
module_a -> module_b -> module_a
```

In this situation, both modules are broken until the circular dependency is fixed (either by reorganizing the code such that the modules don't depend on each other, or by moving one of the dependencies out of the top level). This problem becomes even more difficult when the dependency chain becomes larger. For example, we could have a cyclical dependency looking something like this:

```
# An arbitrarily long dependency chain
module_a -> module_b -> module_c -> ... -> module_n -> module_a
```

In order to fix this problem, which doesn't always manifest itself in an immediate or obvious way [1], an engineer has to find the dependency chain, diagnose the structural issue underlying the dependency chain, and then decide where and how to break the cycle. This is an expensive, frustrating, and ineffective process [2]. 

With that in mind, we can take a look at Khan Academy's webapp:

![Webapp Dependency Tangle](/adr13/webapp_deps.png)
*Each dot in this image represents a module in webapp. Each line represents a dependency between two modules. tl;dr webapp was a tangle (and not even [the cool kind](https://iota.org/IOTA_Whitepaper.pdf))! P.C. Ben Kraft*

As you can see, there wasn't really a coherent logical structure. Our dependency tangle looped back on itself in complicated and frustrating ways. Further, the recurring circular dependency problems we were seeing were indicators of a larger structural issue; webapp had a decided lack of separation between logically distinct parts of the code. This issue with our architecture is also indirectly expensive.

For example, these structural problems

- make it really difficult to hold a cohesive mental image of the codebase [3]
- stand as a barrier to adopting a micro-services architecture
- encourage hacky solutions to avoid circular dependencies [4]
- prevent intentional design of new features
- leave you in the dark as to what downstream side effects even a small code change will entail

### Our Solution

Our solution to the dependency problem was to divide our codebase into about 40 top-level directories (called "packages"), each of which is owned by one of the Khan Academy engineering initiatives. These packages are placed in a hierarchy, where any given package is allowed to depend on packages lower than itself. Any dependency from a lower-level package to a higher-level package can then be classified as a "bad dependency".

By defining what can be depended on by any given code feature, we can automatically detect places in our codebase that are architectually problematic. In short, forcing modularization of our code and enforcing a ban on potentially dangerous dependencies can help us figure out where our codebase is poorly organized and when we need to rethink feature design.

In practice, implementing this solution divided into about three steps:

1. **Defining the packages** --- This meant building a really, _really_ big spreadsheet of every file and every api route in webapp, and then assigning each one to a package. 

2. **Moving each file into its package** --- This is where we spent the bulk of the past two months. Using [slicker](https://github.com/Khan/slicker)--Khan Academy's open source tool for python refactoring--we moved each file from wherever it was living into its proper new home (one of the new directories in the top level of our webapp repo).

3. **Breaking up pre-existing bad dependencies** -- Finally, once we had our package-based architecture in place, we were able to dive into the codebase, find places with a lot of bad dependencies, and try to reorganize and redesign features that were causing problems.

## Hard Things(TM)

Having provided a high-level layout of the problem and our intended solution, we can focus in on the specifics.

Phil Karlton once famously claimed that there are only two hard things in computer science: cache invalidation and naming things. Incidentally, we ran into both of these challenges! Unfortunately, they were not the only challenges we ran into [5]. We can, however, roughly categorize the difficulties we encountered into 4 groups of Hard Things. Here we'll introduce some of the challenges we ran into, and the ways we attempted to solve for them.

### I. Organizing Code

At the beginning of the project, we knew that we wanted everything divided into packages, but we didn't know what those packages would be [6]. The first week or so of the project boiled down to David F., Craig, and distilling every single file and api route in webapp into a monolithic, 2000-line google sheet. As a group, we then looked at each item individually, tried to get to the essense of what it does, and decided where--in which package, that is--it should live.

When we think of categorization, often we think of it in terms of boxes---so analogically, this first week was filled with looking at 2000 different items, and placing them in some number of boxes. However, this particular categorization task was made challenging in that our boxes weren't well-defined. In fact, the definitions of our packages (as well as our definitions and descriptions) changed constantly over the course of developing the file-to-package mapping. Add to this the fact that some of the items we were boxing were old, deprecated, poorly named, and sometimes monolithic and 

Logistically, this meant keeping a lot of things in mind at the same time; rapidly changing definitions and descriptions of an in-flux set of packages, known hubs of depencies, which files relate to which part of the product, which initiative of the eng team is supposed to own each package, etc.

Personally, I found this to be the most challenging aspect of the project. In the way of solutions, there's really no good shortcut. Still though, we did end up with a few successful strategies for this kind of sweeping re-organization of our codebase:

- **Crowdsource decisions**---Having our small team sit down and go through all 2000 files in a big 5-hour meeting (surprisingly, to me at least) proved more effective than paralellizing and trying to divide and conquer. I think this was because the groupthink strategy capitalizes on peoples' unique understandings of different parts of the codebase, and required explicit justification on _why_ something belongs where it does. Both of these helped streamline the evolution 
- **Iterate quickly**---By the time we finished an iteration of "boxing" everything, the boxes had changed, sometimes beyond recognition. It took 2-3 iterations of going through and mapping everything for our set of packages to start to converge. This process is still ongoing, and should continue to improve as our mental image of our the components of our codebase is still resolving.
- **Accept fuzzy processes**---At some point, we had to recognize that not all of the decisions were perfectly clear cut. No matter how rigorous you are with the reorganization process, sometimes the most important observation is that something just doesn't feel right.


### II. Moving Code

The second big part of the project was actually moving our code from wherever it existed before to its new home---a top-level directory representing a packages. To do this, we used a really, really cool tool called slicker. Slicker is a command-line utility for refactoring python code. Slicker is interesting enough that it has its own blogpost, but for the purposes of this article, slicker works by taking the name of a python symbol in your codebase and the location to where you would like to move it. For example, the following would move a python file `old_dir/foo.py` into the `new_dir` directory:

```
slicker.py old_dir/foo.py new_dir/
```

Despite the awesomeness of slicker, there were some nuances that were tough. For example, at the beginning of the project, slicker had a really hard time with common english words. As a result, moving a file such as `user.py` into a package like `accounts` would change any instances of the word "user" in comments and strings to change to the word "accounts.user" (in files that imported the module `user`). This has the potential to become really confusing:

```
# This parent doesn't have a registered child user 
```

would become something confusing (especially if your a developer with no context on the moves)

```
# This parrent doesn't have a registered child account.user
```

More problematically, changing strings can break important functionality:

```
@route("/api/internal/user/profile", methods=["GET"])
```

would be changed to an invalid api route that would keep the site from getting user profiles:

```
@route("/api/internal/accounts.user/profile", methods=["GET"])
```

The big lesson that we learned from this challenge is that investing in automation is almost always worth it. 


### III. Playing Well with Others

One of our concerns going into this project was how to do a sweeping reorganization of our codebase without keeping other projects from getting done or making our inconveniencing our teammates and making our friends angry with us.

For example, consider code reviews. At khan Academy, we use phabricator's Differential to review code, and it's best practice to send small, manageable chunks of code out for review so that we can iterate quickly (**TODO link**). Unfortunately, differential is not always the best at handling large changes across many files[7], and the changes required to refactor webapp were, categorically, cumbersome diffs that moved thousands of lines of code and touched hundreds of files. We called this process of moving all of the code for one component into its new home a "package move", and code reviews for package moves were brutal.

If you request a review on this kind of change from developer who has context on the product but is _not_ working on the refactoring project without providing any context, they're probably (and understandably) going to stare at the screen for a little bit before getting frustrated and confused and resigning from the review.

To make this process easier for everyone, we ended up including as much context and help as possible in our commit messages. Since these get included at the top of each differential review, we would explicitly list some context on the project, the exact steps we took to make the package move (down to the slicker command), the challenges we ran into, and what specific changes we think should recieve the most attention. We even started including jokes at the end of our commit messages as a way to say "I'm sorry you woke up on Monday morning to 70k lines of code to review and thank you for helping"[8].

**TODO screenshots** tl;dr the first picture is much more helpful than the second.

Code reviews aside, there were some other challenges that came with reorganizing the codebase while other developers worked to improve it: there's always a team-wide learning curve when familiar code is split up and rearranged; merge conflicts were gross[9]; we kind of broke `git blame`, which now thinks the three of us doing the package moves are the original authors of most of webapp; etc. In general, we were able to mitigate the inconvenience of these kinds of playing-nice-with-others challenges by being explicit and consistent with our communication and budgeting time to help clean up the messes we made along the way (E.G. making sure to email out the scheduled moves and expecting to spend some flex time helping resolve conflicts when they occur). 


## Outcomes

In closing, we can take a evaluate the efficacy of the project by taking a look at some of the project's outcomes. Here's a highlight reel of some of the wins we've seen:

* **Fewer Bad Dependencies**---We've recorded a measurable improvement in our dependency structure[10]! Having 1) built out a formalized notion of what a bad depenency is and 2) written tools for finding them in our codebase, we've been able to measure the goodness of our dependency structure over the course of the project. When we first took this measurement (before making the moves, and with an entirely hypothetical component structure based on the divisions mandated by our giant monolithic spreadsheet), we found that about 17.5% of all of our inter-package dependencies we bad. On December 13th, after having made all the initial package moves, we took this measurement again as found that 14.6% of our dependencies were bad. Taking this measurement the webapp in production on January 15th, the first day after the project is officially complete, we see that have reduced the percent of bad dependencies in webapp to 7.8%. This improvement represents a series of changes to our codebase's architecture which will streamline the development process and save dozens (maybe even hundreds) of developer-hours every year for the forseeable future! I think this is a really cool result!
* **Better Understanding of Structure**---In addition to getting rid of bad dependencies, we've also taken inventory of the bad dependencies that remain and have tried to understand what work is left to reduce this number even further. We now have a finite set of well-defined tasks that we can do to improve our dependency structure as best as possible. Even though we didn't have time to get to all of these, the work we've done allows us to know what work is left to do, how to do it, and how to prioritize it moving forward.
* **Code Hygeine**---`ls webapp` has been cleaned up significantly[11], and the directory and sub-directory structure of our entire codebase has been audited. This added intentionality with regards to organization makes a big difference. **TODO: Pics**
* **Better Documentation**---For the first time, Khan Academy's webapp has top-level README which serves as a high-level introduction to our product and gives a one-line description of each of our components. In addition, each of our components has a README describing what it does, which engineering initiative owns it, and how it's structured.
* **Clear Mental Model of Webapp**---In with the help of our clean new structure and concise documentation, it's easy to form a clear mental map of our entire codebase and see how different parts of our product relate to one another. This will have a huge impact on the ease of onboarding, but even developers who've been working on webapp for a long time have been been positively impacted and excited by this side effect of ADR-13.
* **Code Ownership**---Every component is explicitly owned by one of khan academy's engineering initiatives. Before ADR-13, there were parts of the product that were a grey area. Now, we don't have to ask "Who owns this, again?" when something breaks or needs to be changed.
* **Potential for Microservices**---The increased modularity of our componentized codebase is a big first step towards a microservice-based architecture. 
* **Potential for Maintenance**---We now have a linter that runs at `commit` time that will not let you add code that either doesn't adhere to component structure or introduces new bad dependencies to webapp[12]. In other words, the improvements introduced over the last milestone will be hard to undo.

Overall, re-architecting webapp with dependency structure in mind was a large and challenging project, but it set up the KA engineering team to work better and more effectively in the future!

Onward, and a happy start to 2018 from the KA team!

## Footnotes

[1] One time I spent 30 minutes.\n

[2] So we usually hack around it and move imports to. Usually, this technique is more of a hacky bandaid on a larger structural issue.

[3] Citation: me

[4] Unused imports eg.

[5] Ironically, neither naming difficulties nor cache invalidation made it to the list of challenges in this article. In short, though, we use absolute imports in our python code, which require that the full paths of imports be specified. So, while `exercise_mechanics` is a descriptive and precise package name in theory, our teammates wouldn't be super excited after calling a static method on a class like `exercise_mechanics.user_simulator.simulated_users.ProbablisticUser`. There are ways around this (`from` imports, assigning the class to a new variable in the file where it's users), but yuck. And the cache invalidation problems we ran into with pickling and unpickling deserve an article of their own.

[6] Although we did have a very well-done rough draft of a set of possible packages and many of the files that would go in them! A big thanks to Ben Kraft for working on this in the months leading up to the TSM - there's no way we could have gotten as far as we did on this project without the groundwork already laid out!

[7] EG one time I ran `arc diff` after creating a particularly large component. After about ~30 minutes of linting and ~20 minutews of unit tests, the diff failed with a "503 Max Size Exceeded Error". (**TODO pic**). To hack around this, I used `arc diff --nolint --nounit --less-context --verbatim` for the rest of my diffs. This hack had the unfortunate side-effect of removing code surrounding my changes, making the diffs harder to review.

[8] Q. What happens to a frog's car when it breaks down?
A. It gets _toad_ away.

[9] Merge conflicts were gross enough that if you didn't deploy a package move within about a day of starting it, you would spend more time fixing merge conflicts than you did doing the actual package move.

[10] Personally, I think this is the coolest of our end results. I'm a really big fan of using new metrics to quantify abstract things like "dependency structure", and measurable improvement is lit.

[11] Fun fact: we found that this unused image has been hiding, entirely unnoticed, in the top level of `webapp` since mid 2015. This strikes me as particularly ironic. **TODO: pic** 

[12] We have a whitelist of all of the bad dependencies that we haven't been able to fix yet. We plan to continue to whittle away at this list in 2018. Developers _could_ theoretically introduce new bad dependencies by adding to this list, but we (the ADR-13 team) are automatically notified by additions to the whitelist and will politely encourage transgressors to find a different solution whenever possible :)
