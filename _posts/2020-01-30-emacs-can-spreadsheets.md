---
title: Emacs can spreadsheets
date: 2020-01-30 10:45
categories: Programming
tags: emacs
---

Emacs is [famously](https://wiki.c2.com/?EmacsAsOperatingSystem) powerful
editor. Its [org-mode](https://orgmode.org) can be used for work logs, literate
programming, or even spreadsheets. In this post I want to outline a specific
feature of this mode: org-table.

Tables in org-mode can have certain cells computed automatically. For that, add
annotations immediatelly after the table, as follows:

```
| 1 | 2 | 3 |
| 3 | 4 | 7 |
#+TBLFM: $3=$2+$1
```

Org-mode provides a simplified syntax for table formulas, which is similar to
what traditional spreadsheet applications offer. For example, in the table
above, the formula populates cells in third column by adding cells from the
first two columns of the same row. In more complicated cases, one
[can](https://orgmode.org/manual/Formula-syntax-for-Lisp.html) write formulas in
Emacs Lisp. Or, even execute external code snippets by calling
[`org-sbe`](https://orgmode.org/worg/org-contrib/babel/intro.html) function. See
the corresponding documentation for more detail.

My use case was a bit more complicated. I had a table where the first column
contains "row names", which I wanted to use as access keys for cell formulas. My
imaginary scenario would look as follows:

```
name_a | 1 | 2 | 4 |
name_b | 3 | 4 | 1 |
#+TBLFM: @name_a$3=@name_a$2+@name_b$1
#+TBLFM: @name_b$3=@name_b$2-@name_b$1
```

Unfortunatelly, org-mode table syntax does not support such addressing. Instead,
I had to combine column formulas with with Emacs Lisp code. Additionally, there
can be only single column formula per column, meaning that I have to put
processing for all the rows into the same formula. This would make formula
extremely long, that quickly becomes unreadable without line breaks and syntax
highlighting. In the end, I came up with following solution:

```
#+NAME: calc
#+begin_src emacs-lisp
  (setq a-map (mapcar* 'cons names va))
  (setq b-map (mapcar* 'cons names vb))
  (defun get-val (map key)
    (setq val (cdr (assoc key map)))
     val)
  (cond ((string= name 'name_a) 
         (+ a (get-val b-map 'name_b) ))
        ((string= name 'name_b) 
         (+ a b))
        ('t cur))
#+end_src

| Name   | a | b | c |
|--------|---|---|---|
| name_a | 1 | 2 | 5 |
| name_b | 3 | 4 | 7 |
#+TBLFM: $4='(org-sbe calc (name '"$1") (a $2) (b $3) (c (string "$4")) (names '(@2$1..@>$1)) (va '(@2$2..@>$2)) (vb '(@2$3..@>$3)));L
```

First, I will start with table formula explanation, then I will describe what
the source code block named "calc" does. The formula is a column-formula for
column 4, which calls elisp function `orb-sbe`. Function `orb-sbe` calls code
block `calc` and passes there a list of arguments. In my case the arguments are
`name`, `a`, `b`, `c`, `names`, `va`, and `vb`. Before running the Emacs Lisp
code, org-mode replaces table variables with values appropriate values. For
second row, `$1`, `$2`, `$3` is replaced with `name_a`, `1`, `2`
correspondingly. Range, `@2$1..@>$1` becomes `name_a name_b`. The formula ends
with `;L`, which says that org-mode should pass values as literals into Emacs
Lisp code (for more details see
[documentation](https://orgmode.org/manual/Formula-syntax-for-Calc.html)).

The code block is written in Emacs Lisp. First, I create two associative arrays
that maps row name to value in each of the columns. Then, I define a
accessor-function to extract values from associative arrays. The block is
finished with cond-expression, which checks row name and returns a value, if it
matches. If no row name matches, the cond-expression returns old value of the
row, because otherwise it will be replaced with `nil`.

Emacs correctly considers this code as secure hazard, because Emacs Lisp can
execute arbitrary code, for this reason it requests user permition to run
formula for each row. Of course this is too annoying, so I had to disable the
confirmation:

```
 # Local Variables:
 # eval: (setq org-confirm-babel-evaluate nil)
 # End:
```

Please use above code at your own risk.

You probably will notice that the code runs relatively slow, because an external
code block is called for each row. Any suggestions to improve performance is
welcome. Of course, the best would be to allow "named rows" in native org-table
formulas, similarly to already existing named columns.
