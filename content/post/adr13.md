+++
title = "Re-architecting for Dependencies"
description = "Untangling Khan Academy's Webapp for Fun and Profit"
date = 2018-01-02
publishDate = 2018-01-02
author = "Carter J. Bastian"
aliases = [
    "/post/adr-13/",
    "/adr-13/",
    "/re-architecting-for-dependencies/",
]
+++

\[ This article is cross-posted from Khan Academy's engineering blog. You can see the original [here](http://engineering.khanacademy.org) \]

For the last two months of 2017, the Khan Academy engineering team has been working on our Technical Sustainability Milestone---an all-hands-on-deck initiative aimed at reducing tech debt and optimizing for effective development in 2018 and beyond. One problem that we decided to tackle during this milestone was dependency management. 

Before the project described here, webapp was a giant, unstructured tangle of dependencies. The lack of dependency structure led to practical and architectural issues which made it difficult not only to build new features, but even to maintain existing ones. 

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

In practice, implementing this solution divided into about steps:

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

The really big lesson that we learned from this challenge is that investing in automation is almost always worth it.


### III. Reviewing Code

### IV. Playing Well with Others

## Outcomes

In closing, we can take a evaluate the efficacy of the project by taking a loot at some of the outcomes we've achieved and where we can go from here.

### Quantifiable Wins

### Onboarding

### Maintained Sustainability
- Posible modularity


## Conclusion


[1] One time I spent 30 minutes.\n

[2] So we usually hack around it and move imports to. Usually, this technique is more of a hacky bandaid on a larger structural issue.

[3] Citation: me

[4] Unused imports eg.

[5] Ironically, neither naming difficulties nor cache invalidation made it to the list of challenges in this article. In short, though, we use absolute imports in our python code, which require that the full paths of imports be specified. So, while `exercise_mechanics` is a descriptive and precise package name in theory, our teammates wouldn't be super excited after calling a static method on a class like `exercise_mechanics.user_simulator.simulated_users.ProbablisticUser`. There are ways around this (`from` imports, assigning the class to a new variable in the file where it's users), but yuck. And the cache invalidation problems we ran into with pickling and unpickling deserve an article of their own.

[6] Although we did have a very well-done rough draft of a set of possible packages and many of the files that would go in them! A big thanks to Ben Kraft for working on this in the months leading up to the TSM - there's no way we could have gotten as far as we did on this project without the groundwork already laid out!
