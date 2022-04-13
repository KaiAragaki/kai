---
title: Serving up plates with {gp}
author: Kai
date: '2022-04-12'
slug: []
categories: []
tags: []
excerpt: "Tidy microwell data using a succinct yet powerful grammar with {gp}"
---

If I'm not staring at a computer screen[^1], I'm staring at a microwell plate, doing my duty as a lab rat: moving little liquids from one place to another.

If you're familiar with this pipettopalooza, you're probably familiar with data that look like these:


```
##      [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8] [,9] [,10] [,11] [,12]
## [1,] 0.07 0.08 0.10 0.12 0.17 0.24 0.38 0.24 0.26  0.25  0.24  0.26
## [2,] 0.07 0.08 0.10 0.12 0.17 0.25 0.40 0.25 0.26  0.26  0.24  0.26
## [3,] 0.07 0.08 0.10 0.13 0.19 0.25 0.40 0.25 0.26  0.26  0.24  0.26
## [4,] 0.27 0.27 0.26 0.27 0.27 0.27 0.26 0.07 0.07  0.07  0.07  0.07
## [5,] 0.25 0.26 0.26 0.26 0.26 0.26 0.25 0.07 0.07  0.06  0.06  0.06
## [6,] 0.25 0.27 0.25 0.26 0.26 0.26 0.25 0.07 0.07  0.07  0.07  0.07
## [7,] 0.08 0.10 0.08 0.08 0.08 0.08 0.08 0.08 0.08  0.08  0.08  0.08
## [8,] 0.05 0.05 0.05 0.05 0.05 0.05 0.05 0.05 0.05  0.05  0.05  0.05
```

This is a mess. For a couple reasons.

1. Only I know what the experimental design is here.
2. Tidying this is going to suck.

One way of going about doing this is to do some slashing and hacking in Excel. I don't want to be too hard on Excel, because it definitely has its place, but frankly I think we can do better: slashing and hacking isn't documented by code (a hit to reproducibility) and if you do this kind of thing a lot, you have to slashack every time (a hit to sanity).

There is, of course, base R, and {dplyr}, and {tidyr}, and some amalgamation thereof, but thinking about doing that tidying process more than once makes me weep. Alternatively, you might use {plater}, a beautiful package by Sean Hughes. One reason you might not choose to is that you again have to do some data munging to get it into the format that is suitable for plater ingestion, which again involves some undocumented steps from the initial raw data file to actually reading stuff in R. 

Enter (after a very tired sales pitch format) {gp}.

# TL;DR

We can simultaneously plot and tidy these data using a succinct yet flexible grammar provided by gp by doing:


```r
my_plate <- gp(8, 12, protein_quant) |> 
  gp_sec("has_sample", 3, 19, wrap = T, labels = "sample") |> 
  gp_sec("replicate", 1, labels = c("1", "2", "3")) |> 
  gp_sec("sample", 3, 1, labels = c(paste0("standard ", 1:7), paste0("sample ", 1:12)))

gp_plot(my_plate, sample)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-2-1.png" width="672" />


```r
gp_serve(my_plate) |>
  head() |> 
  knitr::kable()
```



| .row| .col|  value|has_sample |replicate |sample     |
|----:|----:|------:|:----------|:---------|:----------|
|    1|    1| 0.0691|sample     |1         |standard 1 |
|    4|    8| 0.0739|NA         |NA        |NA         |
|    5|    8| 0.0696|NA         |NA        |NA         |
|    6|    8| 0.0726|NA         |NA        |NA         |
|    2|    1| 0.0693|sample     |2         |standard 1 |
|    3|    1| 0.0711|sample     |3         |standard 1 |

# Plotting is tidying

One of the fun aspects of gp is that you tidy things by essentially creating a plot that matches them. By using the grammar gp provides to essentially 'weave' a plate with all its conditions, it also provides a means to 'unweave' a plate into a tidy dataframe with all its annotations. So first, we plot.

# Making the plate

To begin our plotting journey, we first need to make the plate (before we can begin to describe the plate). 

This is easy!


```r
gp(rows = 8, cols = 12)
```

```
## 
##                 12
##    ________________________
##   | ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯
##   | ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯
##   | ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯
##   | ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯
## 8 | ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯
##   | ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯
##   | ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯
##   | ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯ ◯
```

```
## Start corner: tl
```

```
## Plate dimensions: 8 x 12
```

The printed output gives us some details about the current section, which in this case is the plate. But what do I mean by a section?

# It's sections all the way down

A section is just a rectangular region with a few extra tricks. But in gp, everything is a section. The plate itself - created with `gp()` - is just a special case of a section that doesn't have a parent section[^2] and can take on your data as one of its arguments. But we're getting ahead of ourselves.

What's important is that we build up our plots by layering on sections. Here's an example of adding 3x5 sections on top of our plate:


```r
gp(8, 12) |> 
  gp_sec("example_section", nrow = 3, ncol = 5) |> 
  gp_plot(as.factor(example_section))
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-5-1.png" width="672" />

(Note that I've omitted the argument names in `gp`. I'm going to do the same for `nrow` and `ncol` with `gp_sec` from here out)

Just as I promised, I added 3x5 sections to our plate. `gp_sec()` is the workhorse of gp, and it has a lot of fun arguments that can be used alone or in combination. Let's use one!

## Wrapping

The first argument I'm going to use is `wrap`:


```r
gp(8, 12) |> 
  gp_sec("example_section", 3, 5, wrap = T) |> 
  gp_plot(as.factor(example_section))
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-6-1.png" width="672" />

Note how sections continue onto the rows below instead of being clipped off as before.

## Labels

We can also supply `labels`. Labels give names to individual sections, and additionally any sections without labels are dropped, like so:


```r
gp(8, 12) |> 
  gp_sec("example_section", 3, 5, wrap = T, labels = c("Primer 1", "Primer 2", "Primer 3", "Primer 4")) |> 
  gp_plot(example_section)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-7-1.png" width="672" />

# Snap back to reality

We're now equipped with enough know-how to take on the original plate data at the top of this post. See, I never described what the plate actually represents. It's absorbance readings from 562nm, in order to measure protein concentrations. But not all of the wells have samples. Only these ones do:


```r
gp(8, 12, protein_quant) |>
  gp_sec("has_sample", 3, 19, wrap = T, labels = "sample") |> 
  gp_plot(has_sample)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-8-1.png" width="672" />

In this plating arrangement, there are 19 samples, and each is performed in technical triplicate. Each column represents a sample, each row a technical replicate. The only oddity here is that these samples wrap from one lane of wells down to the next when they hit the edge. 

You may have also noticed that I've included a third argument in `gp`. We _technically_ don't need it until we `gp_serve` our plate, but I'm including it now because that's how you'd probably do it in the wild[^3].

To denote replicates, I can do the following:


```r
gp(8, 12, protein_quant) |>
  gp_sec("has_sample", 3, 19, wrap = T, labels = "sample") |> 
  gp_sec("replicate", nrow = 1, labels = c("1", "2", "3")) |> 
  gp_plot(replicate)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-9-1.png" width="672" />

Couple things to note. First off, once you've excluded part of a parent section by not provided the maximum number of labels (as we did with the `has_sample` section), the remaining wells will never be populated again. They are scorched earth. What was once NA will forevermore be NA. Second, notice how I only included ONE number (`nrow = 1`) and left the other blank. If a dimension isn't specified, it will take up the maximum space possible: the size of its parent section. Here, that's 19.

Finally, I want to denote sample names. It's nothing we haven't seen before:


```r
my_plate <- gp(8, 12, protein_quant) |> 
  gp_sec("has_sample", 3, 19, wrap = T, labels = "sample") |> 
  gp_sec("replicate", 1, labels = c("1", "2", "3")) |> 
  gp_sec("sample", 3, 1, labels = c(paste0("standard ", 1:7), paste0("sample ", 1:12)))
```

I've assigned our `gp` to `my_plate` since we are at journeys end. But, for good measure, let's see what those samples look like: 

```r
gp_plot(my_plate, sample)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-11-1.png" width="672" />

And now, for a final flourish, the tidying part:


```r
gp_serve(my_plate) |>
  head(10) |> 
  knitr::kable()
```



| .row| .col|  value|has_sample |replicate |sample     |
|----:|----:|------:|:----------|:---------|:----------|
|    1|    1| 0.0691|sample     |1         |standard 1 |
|    4|    8| 0.0739|NA         |NA        |NA         |
|    5|    8| 0.0696|NA         |NA        |NA         |
|    6|    8| 0.0726|NA         |NA        |NA         |
|    2|    1| 0.0693|sample     |2         |standard 1 |
|    3|    1| 0.0711|sample     |3         |standard 1 |
|    1|    2| 0.0801|sample     |1         |standard 2 |
|    4|    9| 0.0718|NA         |NA        |NA         |
|    5|    9| 0.0667|NA         |NA        |NA         |
|    6|    9| 0.0727|NA         |NA        |NA         |

# Conclusion

There's a lot I didn't cover here for the sake of space and your sanity. If you want to learn more, check out the  [`gp` pkgdown site](https://kaiaragaki.github.io/gp), particularly the ['Making Plots with gp'](https://kaiaragaki.github.io/gp/articles/making-plots-with-gp.html) article. If you have any questions, feel free to reach out to me on [twitter](https://twitter.com/kaioinformatics), or open an issue on [GitHub](https://github.com/KaiAragaki/gp/issues).

# Acknowledgements

[Hermes Rivera](https://unsplash.com/@hermez777) took the beautiful picture of salad. 

Thank you to my mentor, Dr. David McConkey, for not only allowing, but encouraging me to pursue these side projects.


[^1]: Or rather, trying to peer through my cat, presumably protecting me from looking at my hideous code

[^2]: More technically, a `gp()` is its own parent

[^3]: Actually, there's an even slicker way to do this: `as_gp`, which automatically sets `gp` plate dimensions based on the dimensions of the input data. This has the advantage of being readily pipeable.
