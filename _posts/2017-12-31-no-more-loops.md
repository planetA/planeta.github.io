---
title: No more loops
date: 2017-12-13 19:09
categories: Programming
tags: R hack
---

I like writing in R and often I do vizualisations using ggplot2. I
often have a need to generate a multi-page PDF document from several
disjoint groups in a single data frame. In dplyr groups are created
using `group_by()` function. The data.table package provides special
`by`-syntax. Grouping in R is analagous to grouping in SQL. After
doing the computations it is often useful to visualize the result for
each group separately. For that ggplot provides functions `facet_grid`
and `facet_wrap`. Syntax looks as follows.

```R
ggplot(df, aes(x, y)) + geom_line() + facet_wrap(~z)
```

Unfortunatelly, if the results are to be printed into a PDF document,
all the plots are going to be on the same page. This is not always
what I want. Previously I was creating a loop around ggplot command,
selecting subset of rows for each dataframe and printing groups one by
one. The code always looked ugly. But recently I discovered for myself
a simple and neat way to get rid of the loop. If you are using dplyr,
you can benefit from combining `group_by`, `do` and `ggplot` together.

The resulting code will look something like this:

```R
p <- df %>%
    group_by(a, b, c) %>%
    do (
        plots = ggplot(data = .) +
            geom_line()
    )
print(p$plots)
```

The data frame is spilt into groups and then `do` applies a function
to each group. The function `do` has to return a dataframe, that's why
one has to give a name to a series of plots. There is a variable `.`,
created by dplyr, which represents the argument piped into to
`do`. For example, to get a single column of data, one would need to
write `.$col`. The last line applies print to the whole series of
plots one by one.

