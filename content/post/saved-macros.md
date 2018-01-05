+++
title = "Recycling Vim Macros"
description = "A semi-automated approach to lint errors"
date = 2018-01-04
publishDate = 2018-01-04
author = "Carter J. Bastian"
aliases = [
    "/post/recycling-vim-macros/",
    "/recycling-vim-macros/"
]
+++

[Note: this article assumes that you, the reader, have some knowledge of vim, vim commands, and vim macros. If you're not familiar with any of this, I would highly recommend (*TODO: find vim my old vim source*)]

Vim macros are neat [1]. The ability to write, store, and repeat arbitrary operations by doing them in the editor once is one of the primary reasons I use vim for (almost) everything I write.

Lint errors [2], on the other hand, are not neat.

In the project my team has been working at Khan Academy for the past few months, we've been doing a _lot_ of code reorganization and moving files around. To move make this big moves effectively (without breaking anything, that is), we've been using an internally-developed but entirely open-source tool for python refactoring called slicker (*TODO: Link*). 

Now, slicker is smart and gets most transformations (semantically) correct, but it's not _quite_ smart enough to reliably break long lines in a pythonic and aesthetic manner [3]. As such, moving commonly-used and widely-referenced files with slicker sometimes results in 100s of lint errors.

This article is going to use the example of resolving slicker-generated lint errors to illustrate how developers can very quickly automate redundant tasks by saving commonly used operations in vim. While the example is fairly specific, the underlying principle is that proper use and reuse of vim macros can save a lot of time with little overhead (*TODO: link*).

<br />
# An Example Problem

Let's consider a hypothetical. Consider a repository with two files:
```bash
> tree
.
├── commonly_used_module.py
└── foo.py
```

Here, `commonly_used_module.py` is some file with a bunch of code that you import and use in `foo.py`. For illustration, let's say that `foo.py` looks like this:
```python
# foo.py 
import commonly_used_module

my_long_variable = commonly_used_module.ImportantClassName(param_1,
                                                           param_2,
                                                           param_3)
```

Now suppose that we want to move `commonly_used_module.py` into a new subdirectory called `package/`. To do this, we would run something like this: 

```bash
> slicker.py commonly_used_module.py new_directory/
> tree
.
├── foo.py
└── new_directory
    └── commonly_used_module.py
```

During this change, slicker would move `commonly_used_module.py` into `new_directory/` and update all the places that import and use the module we moved. Afterwards, then, `foo.py` would look like this:

```python
# foo.py 
import new_directory.commonly_used_module

my_long_variable = new_directory.commonly_used_module.ImportantClassName(param_1,
                                                           param_2,
                                                           param_3)
```

Syntactically and semantically, this is correct, but now we can see a couple of stylistic problems:

1. Line 4 is longer than 79 characters. According to PEP8, this is too long.
2. The paramters on lines 5-6 are not indented so that they're aligned with the parameter on line 4.

Sure enough, running the Khan Linter (*TODO: link*), we get the following output:
```
foo.py:4:80: E501 line too long (81 > 79 characters)
foo.py:5:60: E128 continuation line under-indented for visual indent
foo.py:5:60: E128 continuation line under-indented for visual indent
```

<br />
# A Recycled Macro Solution

Fixing the lint error above by hand is totally reasonable... once. However, if we were moving code in a real-life codebase, this class of error would be likely to appear multiple times and in multiple files. Solving this recurring lint error without some form of automation quickly becomes tedious, and sometimes becomes infeasible<sup>[4]</sup>. 

That's where our recycled vim macros come in. After hitting this same class of lint errors a few times, I recognized that I was typing the same few patterns over and over and over again. I started saving these patterns as macros whenever I would head into a session of solving lint errors. Finally, I got fed up with re-writing and saving my macros every time I opened a file with lint errors, so I put the following lines in my `.vimrc` file[5]:

```vimscript
" Macros  
let @q='0f(a^M^[j'         " Macro #1 stored in register q
let @w='0^d0i^?^M^[j'       " Macro #2 stored in register w
let @e='k0^d0i^? ^[j'       " Macro #3 stored in register e
```

Let's break these lines down a bit. Let's start with Macro #1:


* `let @q='...'` --- This says, "store the string `...` as the macro in register `q`
* `0f(` --- `0` takes us to the beginning of the current line, and `f(` takes us to the first opening parenthesis
* `^M^[` --- These are the escape characters[6] for the enter key and the escape key respectfully. Since we're in insert mode, this presses enter and then escape, leaving us in normal mode again
* `j` --- This moves us down to the next line

In total, then, this line saves a macro to register `q`[7] which inserts a new line after the first opening parenthesis on the current line, leaving your cursor on the line below the line you started with.

<br />
# The complete picture


<br />
# Conclusion

Elephant in the room, there are better ways to automate this kind of thing[?]. If this problem of consistently running into large amounts of these specific lint errors were a problem I was going to continue to see, I would need to find a better way. As it were, the amount of time I would have needed to automate this further was turning out to be more time than I spent fixing errors with the method above. I'm sure that, with enough yak shaving, I could find a way to reduce the amount of time I spend with lint errors by another magnitude, but in this case, that wasn't a rabbit hole worth going down and I was very happy to have a low-effort option that allowed me to (semi) automate the process.

<br />
## Footnotes
[1] This is an understatement. A more accurate statement would be "vim macros are so neat that, paired with a bit of vim-fu, I completely avoided learning even basic commandline utilities for an entire CS degree, and I should not be trusted with any tool as neat as vim macros."
<br />
[2] *TODO: Explain linter and link to KA style guide*
<br />
[3] To be fair, I'm not smart enough to reliably break long lines in a pythonic and aesthetic manner either.
<br />
[4] One of the bigger moves that I worked on this most recent Khan Academy project created more than 500 lint errors spreading across about 300 files, and all of the lint errors were almost exactly like this example.
<br />
[5] Ok, well, not these lines exactly. My [actual `.vimrc` file](https://github.com/carterjbastian/vimrc) has a couple nifty remappings, including one which replaces the escape key. The listed version is the version of my macros that would work on straight-out-of-the-box vim.
<br />
[6] In this case, `^M` isn't the two characters `^` and `M` in sequence. Rather, it's a special single character representing the delete key. You can enter in in insert mode by pressing `Ctrl-v` followed by the delete key. Same goes for `^[` and the Escape key.
<br />
[7] These specific registers were chosen totally arbitrarily. Really they're just close to the `@` key and next to each other. They could be saved to any valid register.

[?] And I'd love to hear what they are from you! Comment below to share yours!
