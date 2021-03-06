# Introducing `plyrmr`
Antonio Piccolboni  










## The making of `plyrmr`

You may already be familiar with the RHadoop project, an open source project which has produced several R packages that interface R with different components of Hadoop. `plyrmr` is the latest in the series and it allows to perform a number of data manipulations and summarizations using an API that borrows ideas from the very popular package `plyr`, SQL and its predecessor, `rmr`. `rmr` is still being developed and covers a wider set of use cases; with `plyrmr` we are focusing on structured data, such as that stored in a data frame, and we are aiming for improved ease of use. We identified 4 aspects of the `rmr` API that could be improved upon:

* The widespread need to define functions to be passed as arguments, even for run-of-the-mill operations such as taking the ratio of two columns; in the style of `plyr`, many `plyrmr` functions accept expressions that are evaluated in the context of the data.
* The requirement that such functions always accept and return two data structures, one containing the data itself and one defining how it should be grouped &mdash; key and value in mapreduce jargon; in `plyrmr`, any user-supplied functions accept a data frame and return a data frame
* The adoption of a SQL-like primitive `group` to replace the mapreduce notion of key-value pairs; it should be familiar to most and stays out of the way when grouping is unimportant or not being acted upon; grouping can be refined, coarsened, reset or just left alone.
* The adoption of delayed evaluation of `plyrmr` expressions allows for the implementation of optimization techniques that reduce the cost of abstraction and encourages reuse of `plyrmr`-based functions.
* While `rmr` is a foundational package with a minimal API, the `plyrmr` API is more in the camp of so-called [humane API](http://martinfowler.com/bliki/HumaneInterface.html), with common use cases captured by separate function calls, even when the implementation is little more than a one-liner.

## First steps

Since code is worth a thousand words, let's get introduced to `plyrmr` through simple examples. We will work with a well known data set, `mtcars`, which certainly doesn't have the size to justify the use of Hadoop but is comes handy for an introduction (`plyrmr` inherits from `rmr` a local backend whereby you can try and learn almost all features without even installing hadoop). 

A first step would be to define a new column that holds the ratio of two other columns. We can accomplish that using the function `bind.cols`, which is a variant of base::transform and `plyr::mutate`, but has advanced features that are useful for programming and a more reasonable name.


```r
bind.cols(mtcars, carb.per.cyl = carb/cyl)
```

We can see the pattern of classic base functions `transform` and `subset`, extended into a complete DSL and popularized by `plyr`: a function name that identifies a broad but related set of data transformations, a data argument and then a number of R expressions, evaluated in an environment expanded with the columns of the data.
This is an in memory, small-data, sequential operation. How quaint! What if we wanted to perform exactly the same operation on a Hadoop-sized data set? The data is in a HDFS directory, hosted on multiple disks and machines. Processing it on a single processor would be unbearably slow. We sure need to become experts in parallel, distributed programming to take a stab at this, right? Wrong. Hadoop changes that and we get most of the Hadoop goodness with `plyrmr`, without much Hadoop jargon at all.


```r
bind.cols(input("/tmp/mtcars"), carb.per.cyl = carb/cyl)
```

Let's review what this does. We passed an HDFS path to the function `input`. This returns an object that represents the whole data set, but doesn't hold it in memory, not even a fraction of it. This type of object can be used as the data argument to any `plyrmr` function. After that, we call `bind.cols` as usual. What happens behind the scenes is that the operation is performed in parallel in the hadoop cluster, by calling the `bind.cols` data frame function on reasonably-sized chunks of data. What gets printed as a result is only a sampling of the rows, the output of the print method on a data object. If we want to bring all the data in memory as a regular data frame, we can just call `as.data.frame` as follows:


```r
as.data.frame(bind.cols(input("/tmp/mtcars"), carb.per.cyl = carb/cyl))
```

This expression is not scalable and will fail for large data sets. You would call something like this after one or more steps of filtering and summarization have reduced the data size to something that can fit in memory. We may then use the returned data frame in the usual way, for instance to produce a plot.
If we need to store the data for later use, we need an additional primitive, `output`, which takes a data object returned by most `plyrmr` functions and writes it to the desired path.


```r
output(
	bind.cols(
		input("/tmp/mtcars"), 
		carb.per.cyl = carb/cyl), 
	"/tmp/mtcars.out")
```


## Predefined operations

As previously mentioned, `plyrmr` aims to provide a wide palette of predefined operations to cover basic data manipulation needs. Here's the list as of version 0.3.0, but still a work-in-progress:


|group|function|
|----|----|
|data manipulation| `bind.cols`, `transmute`, `where`, `select`, `rbind`|
|summaries| `transmute`, `sample`, `count.cols`, `quantile.cols`, `top.k`, `bottom.k`|
|set operations|`union`, `intersect`, `unique`, `merge`|
|`reshape2`| `melt`, `dcast`|

`transmute` appears twice because it's a generalization over transform and summarize that allows to increase or decrease the number of columns or rows, covering the need for multi-row summaries, flattening of data structures and more. In fact we may split its functionality in the future. melt and dcast are Hadoop versions of the originals in the package reshape2 and allow to change the format of a data set from wide to tall and vice-versa. Hopefully the other names are self-explanatory. All these functions return data objects and are scalable (often at the cost of some approximation)

## Combining operations

Any good API provides basic elements, ways to modify them and ways to combine them. This way a wide variety of computations can be described in a language concise enough for people to understand. `plrymr` is no different in that all of the above functions can be freely combined. In the following example, we see how to create a new column and then apply a filter based on that column.


```r
where(
	bind.cols(
		input("/tmp/mtcars"),
		carb.per.cyl = carb/cyl),
	carb.per.cyl >= 1)
```

The bonus feature is that each call, run in isolation, requires a mapreduce job, that is a complete pass of the data. Combined, they require only one. This is the kinds of optimizations allowed by delayed evaluation.

Deeply nested expressions can become hard to read since the name of the functions can be arbitrarily far from all but one of its arguments. Some sort of "syntax locality" principle is violated. To address that we can adopt two alternate styles. The first involved assigning intermediate results to temporary variables, as in


```r
x =	bind.cols(mtcars, carb.per.cyl = carb/cyl) 
where(x, carb.per.cyl >= 1)
```

The second latches onto a recent trend of introducing a Unix-like pipe operator in R (see packages `vadr`, `dplyr` and `magrittr`). 


```r
mtcars %>%
	bind.cols(carb.per.cyl = carb/cyl) %>%
	where(carb.per.cyl >= 1)
```

Whatever the syntax, the resulting computations is the same &mdash; choose the style that you find most suitable. In the following, you will see quite a bit of the Unix style for all but the simpler expressions.

##  Custom operations

What if the built in functions and their combination doesn't cover what you are trying to do? There is another possibility, which is to take a regular data frame function, meaning one that accepts and returns a data frame, and promote it to a Hadoop-capable function. For instance, let's say we need to grab the last column of a general data frame, that is we don't know the name or position of the column in advance. That's easy to do sequentially and in-memory with 


```r
last.col = function(x) x[, ncol(x), drop = FALSE]
```

Enter the function `gapply`. The "g" stand for "group", and we will talk more about groups shortly. The idea is that a Hadoop data set will be acted upon in chunks that fit memory. Until now, we haven't introduced a way to affect those chunks, so consider them as arbitrary groups. `gapply` will take a data object and data frame function, apply the function, using Hadoop, to every group, and collect the results.


```r
gapply(input("/tmp/mtcars"), last.col)
```

It will do exactly what you expect for simple functions that can be applied to each chunk of data independently. For instance, selecting a certain column fits this profile, as does filtering rows based on the content of each row only. But if, for instance, you wanted to extract the first few rows of a data set with `head`, and combined that with `gapply`, you would end up with a fairly arbitrary selection of rows. That's where the art of mapreduce programming kicks in.

## Grouping

If all processing must happen in groups, we need to be able to define and modify them, and the `group` function and its relatives come to the rescue. `group` takes a data object as first argument and then one or more expressions to be evaluated in the context of the data. The data will be split based on the values of those expressions, exactly like in SQL group statement. At this point the grouped data set can be passed to any other `plyrmr` function which will act on the groups. For instance, if one wanted to compute a per-group average, where the groups are defined by the column `cyl`, this is the expression that does it:


```r
input("/tmp/mtcars") %>%
	group(cyl) %>%
	transmute(mean.mpg = mean(mpg))
```

Of course, it's not all roses. Groups need to be sized so as to fit main memory, unless the summary operation has some favorable properties so that it can be applied even without loading a group in full. Very small groups, say a few rows each, would make for a very inefficient program as an R function needs to be called for each group. We are working on ways to chip away at these limitations without introducing too much complexity for the user.


## From data manipulation to statistics

To cap this introduction, a couple of examples that point at more advanced applications of this library. As mentioned before, multiple row summaries are possible in `plyrmr`, unlike, for instance, SQL or `plyr`. This fits very well with the notion of statistical summary, such as a series of quantiles. The quantile.cols computes empirical quantiles of the distribution of each numeric column. It will act on each group of the data set, or on the whole data set if it isn't grouped. It will work on very large groups as well, switching to an approximate mode. Its implementation uses  other `plyrmr` functions such as `gapply` and gather (a special case of group) and is also intended as an example of how to build on top of `plyrmr` and extend it. 


```r
input("/tmp/mtcars") %>%
	group(carb) %>%
	quantile.cols() 
```

And finally, an excursion into modeling. `plyrmr` is not specifically for modeling, but there's nothing preventing you from combining the trove of R modeling functions with it. You just pretend a model is a summary like any other, which may or may not stretch your definition of a summary. The only technicality is to wrap the `lm` call in a list call, so as to produce a value of length one; otherwise `plyrmr` will create a row for each element of a linear model, which gets confusing pretty quickly.


```r
models = 
	input("/tmp/mtcars") %>%
	group(carb) %>%
	transmute(model = list(lm(mpg~cyl+disp))) %>%
	as.data.frame()
models
```

This was just a sampler: please see the full [tutorial](https://github.com/RevolutionAnalytics/plyrmr/blob/master/docs/tutorial.md) and other documentation on the [wiki](https://github.com/RevolutionAnalytics/RHadoop/wiki/user-plyrmr-Home). We also have a [forum](https://groups.google.com/forum/#!forum/rhadoop) where you can ask questions. Talk to you soon!
