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


Vim macros are neat<sup>[[1]](#FN1)</sup>. The ability to write, store, and repeat arbitrary operations by doing them in the editor once is one of the primary reasons I use vim for (almost) everything I write.

Lint errors<sup>[[2]](#FN2)</sup>, on the other hand, are not neat.

In the project my team has been working at Khan Academy for the past few months, we've been doing a _lot_ of code reorganization and moving files around. To make these big refactors effectively (without breaking anything, that is), we've been using an internally-developed but entirely open-source tool for python refactoring called [slicker](https://github.com/Khan/slicker). 

Now, slicker is smart and gets most transformations (semantically) correct, but it's not _quite_ smart enough to reliably break long lines in a pythonic and aesthetic manner[<sup>[3]](#FN3)</sup>. As such, moving commonly-used and widely-referenced files with slicker sometimes results in tens or even hundreds of lint errors.

This article is going to use the example of resolving slicker-generated lint errors to illustrate how redundant tasks can be quickly and organically automated by saving commonly used operations in vim. While the example is fairly specific, the underlying principle is that proper use and reuse of vim macros can save a lot of time with little overhead.

## Before we get started
I'm going to make a couple of assumptions in this post: 

* I assume that you have some knowledge of vim, vim commands, and vim macros. If that's not the case, I would highly recommend [Mastering Vim Quickly](https://jovicailic.org/mastering-vim-quickly/).
* I assume that you have seen your `.vimrc` file before and know how to edit it. If you haven't, I would highly recommend [Learn Vimscript the Hard Way](http://learnvimscriptthehardway.stevelosh.com/)
* I assume that your vim setup already plays nicely[<sup>[4]](#FN4)</sup> with python (i.e. that you have filetype-dependent indenting on and that your filetype-dependent indentation indents python correctly).

<br />
# An Example Problem

Let's consider a hypothetical. Consider a repository with two files:
```bash
> tree
.
├── commonly_used_module.py
└── foo.py
```

Here, `commonly_used_module.py` is some file with a bunch of code that you import and use in `foo.py`. Let's say `foo.py` looks something like this:
```python
# foo.py 
import commonly_used_module

my_long_variable = commonly_used_module.ImportantClassName(param_1,
                                                           param_2,
                                                           param_3)
# Etc...
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

During this change, slicker would move `commonly_used_module.py` into `new_directory/` and update any mention of the old module name. After this, slicker would have changed `foo.py` to look like this:

```python
# foo.py 
import new_directory.commonly_used_module

my_long_variable = new_directory.commonly_used_module.ImportantClassName(param_1,
                                                           param_2,
                                                           param_3)
# Etc...
```

Syntactically and semantically, this is correct. However, there a few stylistic problems:

1. Line 4 is longer than 79 characters. According to [PEP8](https://www.python.org/dev/peps/pep-0008/), this is too long.
2. The paramters on lines 5-6 are not aligned with the parameter on line 4. Yuck!

Sure enough, running our linter, we get the following output:
```
foo.py:4:80: E501 line too long (81 > 79 characters)
foo.py:5:60: E128 continuation line under-indented for visual indent
foo.py:5:60: E128 continuation line under-indented for visual indent
```

<br />
# A Recycled Macro Solution

Fixing the lint errors above by hand is a reasonable thing to do... once. However, if we were moving code in a real-life codebase, this class of error would be likely to appear multiple times and in multiple files. Solving this recurring lint error without some form of automation quickly becomes tedious, and sometimes becomes infeasible[<sup>[5]](#FN5)</sup>. 

That's where our recycled vim macros come in. After hitting this same class of lint errors a few times, I recognized that I was applying a few patterns of operations over and over and over again. I started saving these patterns as macros whenever I would head into a session of solving lint errors. I eventually got fed up with re-writing and saving my macros every time I opened a new vim session, so I put the following lines in my `.vimrc` file<sup>[[6]](#FN6)</sup>:

```vimscript
" Macros  
let @q='0f(a^M^[j'          " Macro #1 stored in register q
let @w='0^d0i^?^M^[j'       " Macro #2 stored in register w
let @e='k0^d0i^? ^[j'       " Macro #3 stored in register e
```

Let's break these lines down a bit. In all three cases, the form of the `.vimrc` line is
```vimscript
let @X='<OPERATIONS>'
```

This saves the operation completed by typing the keys in the single quotes (`<OPERATIONS>`) to some register `X`<sup>[[7]](#FN7)</sup>. Because we're setting the values of these registers in our `.vimrc` file, they'll be available whenever we open a file with vim! 

Let's go through what each macro does, starting with **Macro #1**:

```vimscript
let @q='0f(a^M^[j'          " Macro #1 stored in register q
```

* `0f(` --- `0` takes us to the beginning of the current line, and `f(` takes us to the first opening parenthesis
* `a^M^[` --- `a` puts us in insert mode starting after the character under the cursor. `^M` and `^[` are the escape characters<sup>[[8]](#FN8)</sup> for the enter key and the escape key respectfully. Since we're in insert mode, this is the equivalent of pressing the enter key and then the escape escape key. This leaves us in normal mode again
* `j` --- Moves the cursor down to the next line

In total, then, macro #1 inserts a new line after the first opening parenthesis on the current line, leaving your cursor below the line where it started.

As for **Macro #2**:

```vimscript
let @w='0^d0i^?^M^[j'       " Macro #2 stored in register w
```

* `0^` --- Navigates the cursor to the first non-whitespace character on the current line
* `d0` --- Deletes everything from the beginning of the line up to the cursor
* `i^?^M^[` --- `i` enters insert mode before the character under the cursor. `^?` is the escape character for the delete key (equivalent to pressing backspace once). `^M^[` enters and escapes as in macro #1

Overall, this macro deletes the indentation before the current line and replaces it with the indentation suggested by your filetype indent settings. For reference, I use [pymode](https://github.com/python-mode/python-mode), which gets it right almost all of the time<sup>[[9]](#FN9)</sup>.

Finally, **Macro #3**:

```vimscript
let @e='k0^d0i^? ^[j'       " Macro #3 stored in register e
```

* `k` --- Moves the cursor to the previous line<sup>[[10]](#FN10)</sup>
* `0^d0` --- Deletes leading whitespace (as per macro #2)
* `i^? ^[` --- Enters insert mode, deletes one character, and adds a space. Then escapes back to normal mode
* `j` --- Moves the cursor down to the next line

So all this macro does is join the content of the previous line with the contents of the current line (replacing all whitespace between the two lines with a single space).

It might not be clear how these macros will help with our lint problem, so let's go through an example of how'd we use them! Suppose that we start with our poorly-styled `foo.py` from before.

```python
# foo.py 
import new_directory.commonly_used_module

my_long_variable = new_directory.commonly_used_module.ImportantClassName(param_1,  # Cursor starts on this line
                                                           param_2,
                                                           param_3)
# Etc...
```

If we start with our cursor on line 4 (the line with >79 characters), we can fix the line-too-long error by breaking the line at the opening parenthesis of the function call. Typing `@q` while our cursor is on line 4 of our poorly-styled file, we get this:
```python
# foo.py 
import new_directory.commonly_used_module

my_long_variable = new_directory.commonly_used_module.ImportantClassName(
    param_1,  
                                                           param_2, # Cursor on this line
                                                           param_3)
# Etc...
```

Now lines 4-5 look correct. With our cursor already on line 6, we can align `param_2` with `param_1` on line 5 by typing `@w`:

```python
# foo.py 
import new_directory.commonly_used_module

my_long_variable = new_directory.commonly_used_module.ImportantClassName(
    param_1,
    param_2,
                                                           param_3)  # Cursor on this line
# Etc...
```

Now we have out parameters lined up, which is technically correct. Stylistically, though, I prefer having multiple short params on a single line. Since there's space for `param_2` on the same line as `param_1`, we can type `@e` (with our cursor on line beneath `param_2`) to get `param_2` up to line 5:
```python
# foo.py 
import new_directory.commonly_used_module

my_long_variable = new_directory.commonly_used_module.ImportantClassName(
    param_1, param_2,
                                                           param_3)  # Cursor on this line
# Etc...
```

Finally, we can repeat the steps we took for `param_2` by typing `@w@e` again. This will align `param_3` and then move it up to the same line as `param_2`. We are left with the following file that has no lint errors:

```python
# foo.py 
import new_directory.commonly_used_module

my_long_variable = new_directory.commonly_used_module.ImportantClassName(
    param_1, param_2, param_3)  
# Etc...  # Cursor on this line
```

So, starting on the first problematic line, we used the keystroked `@q@w@e@w@e` to solve all of our lint errors, and we didn't even have to navigate or worry about our cursor position!

<br />
# Bringing it all together

When you really break down this process, it seems way more complex and artificial than it feels when applied. I would argue that the process of breaking the macros down and explaining them was far more difficult (and *way* more time consuming) than the process of actually creating them and putting them to use. The latter process felt pretty organic and natural. I noticed that I was repeating the same operation to fix a recurring error, so I recorded a macro to do the operation for me. After recording the same macro a couple of times, I put it in my `.vimrc` and used it (without really having to think about it) whenever I saw a familiar problem.

Overall then, my lint-fixing workflow ended up looking something like this<sup>[[11]](#FN11)</sup>:

* For each file with lint errors:
    * Open the file
    * Move to the first problematic line by searching `\/\%>79v.\+`
    * While there are lines with more than 79 characters:
        * Use macro #1 (`@q`) to fix the long line
        * Apply macros #2 and #3 (`@w` and `@e`) as needed to fix indentation problems
        * Move to the next problematic line by pressing `n` to get to the next search result

This wasn't a perfect process---not every lint error was solvable with these macros and I had to solve certain edge cases by hand (or with other macros). That said, I would estimate that this semi-automated process streamlined about 90% of the lint errors, and decreased the amount of time I spent trying to get the linter to pass by at least an order of magnitude.

Elephant in the room, there are probably better ways to automate this kind of thing<sup>[[12]](#FN12)</sup>. If this problem were a problem I was going to continue running into, I would need to find a better way. As it were, the amount of time I would have needed to automate this further was turning out to be more time than I spent fixing errors with the method above. 

I'm sure that, with enough [yak shaving](http://www.catb.org/~esr/jargon/html/Y/yak-shaving.html), I could find a way to reduce the amount of time I spent on lint errors by another order of magnitude. In this case, that wasn't a rabbit hole worth going down and I was very happy to have a low-effort semi-automated option.

<br />
## Footnotes
<span style="white-space:nowrap" id="FN1">[1]</span>
This is an understatement. A more accurate statement would be "vim macros are so neat that, paired with a bit of vim-fu, I completely avoided learning even basic command line utilities for an entire CS degree, and I should not be trusted with any tool as neat as vim macros."

<span id="FN2">[2]</span>
At Khan Academy, we have a [python linter](https://github.com/Khan/khan-linter) that is run at commit-time, before sending off a code review, and on deploy. This linter inforces Khan Academy's [python style guide](https://github.com/Khan/style-guides/blob/master/style/python.md).

<span id="FN3">[3]</span>
To be fair, I'm not smart enough to reliably break long lines in a pythonic and aesthetic manner either.

<span id="FN4">[4]</span>
Hint: see [this guide](https://wiki.python.org/moin/Vim) to figure that out!

<span id="FN5">[5]</span>
One of the bigger moves that I worked on this most recent Khan Academy project created more than 500 lint errors spreading across about 300 files, and all of the lint errors were almost exactly like this example.

<span id="FN6">[6]</span>
Ok, well, not these lines exactly. My [actual `.vimrc` file](https://github.com/carterjbastian/vimrc) has a couple nifty remappings, including one which replaces the escape key. The listed version is the version of my macros that would work on straight-out-of-the-box vim.

<span id="FN7">[7]</span>
These specific registers were chosen totally arbitrarily. Really they're just close to the `@` key and next to each other. They could be saved to any valid register.

<span id="FN8">[8]</span>
In this case, `^M` isn't the two characters `^` and `M` in sequence. Rather, it's a special single character representing the enter key. You can enter it in insert mode by pressing `Ctrl-v` followed by the enter key. Same goes for `^[` and the Escape key.

<span id="FN9">[9]</span>
The one scenario I've found where pymode gets it wrong (at least as per the KA Style Guide) is in statements like `class MyClass(ParentClass):`. If we break after the opening parenthesis, pymode will leave `ParentClass` indented once (which is the same indentation as the content of the class) when it should be indented twice. I hit this edge case enough that I ended up saving a modified version of macro #2 which adds the extra indentation.

<span id="FN10">[10]</span>
I did this because, in practice, I ended up always using macro #3 right after using macro #2, which leaves your cursor beneath the line you re-indented. As per the Spirit of Vim, I hate navigating. These macros were designed so that the cursor positioning is consistent and I never have to change it myself.

<span id="FN11">[11]</span>
This is the process for long line errors. For under/over indentation errors, the process was slightly different and involved a bit of bash scripting to open each file on the line with the problematic indentation. Then all I had to do was use macro #2 to re-indent.

<span id="FN12">[12]</span>
And I'd love to hear what they are from you! Comment below to share yours!
