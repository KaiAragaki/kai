---
title: Surviving without survminer
author: "Kai"
date: '2021-12-15'
slug: []
categories: []
tags: []

---

I make a lot of Kaplan-Meier curves. They're pretty common in the field of cancer genomics, and perhaps for good reason: They display a lot of information at once.

For quite a while, I've been using the `survminer` package to make my plots. It's a handy package that can take on a variety of fits and produce a `ggplot`, as well with a truckload of arguments to help you easily customize your figures. As an example:


```r
library(survival)
library(survminer)
library(tidyverse)
library(knitr)
# devtools::github_install("McConkeyLab/bladdr")
library(bladdr)

fit <- survfit(Surv(time, status) ~ sex, data = lung)

ggsurvplot(fit)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-1-1.png" width="672" />

`ggsurvplot` does a lot of heavy lifting for us, including coercing the `fit` object into something actually plottable. As I mentioned, there's a lot of customization you can do just by supplying arguments to `ggsurvplot`:


```r
ggsurvplot(fit, 
           pval = TRUE, 
           risk.table = TRUE, 
           palette = viridis::plasma(2, end = 0.8))
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-2-1.png" width="672" />

I could go on. Generally, if there's something you want changes, there's an argument for that.

# The problem

Survminer does have a few issues here and there, and unfortunately does not seem to be actively maintained any longer (as of writing the last commit was March 9th - it currently has 197 open issues, and 279 closed. Additionally, the package author's commit history, Alboukadel Kassambara, seems to have dropped off in June 2020). And that's ok! But we should think about what to do in the meantime.

I'm working on plotting a bunch of different datasets that I'd like faceted. `survminer` has `ggsurvplot_facet` for this:


```r
library(dplyr)

# I'm doing things that will likely make statisticians cry. 
# This is just an example!

mye_2 <- myeloid |> 
  select(sex,
         time = futime,
         status = death)

lung_2 <- lung |> 
  mutate(status = status - 1, # Originally coded as 1 = censored, 2 = dead
         sex = ifelse(sex == 1, "m", "f"))


combined <- 
  bind_rows(lung = lung_2,
            myeloid = mye_2,
            .id = "id") |> 
  as_tibble() |> # For stripping rownames
  select(time, status, sex, id)

fit_facet <- survfit(Surv(time, status) ~ sex + id, data = combined)

ggsurvplot_facet(fit_facet, facet.by = "id", data = combined)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-3-1.png" width="672" />

Excellent! But what if we were interested in the RELATIVE difference in survival between males and females, rather than the overall survival? This plot shows that individuals in our lung dataset tend to die more rapidly than in our myeloid dataset, but it doesn't highlight the relative death of males vs females in the context of the disease. One way to do this is to free our x-scale:


```r
ggsurvplot_facet(fit_facet, data = combined, facet.by = "id", scales = "free_x")
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-4-1.png" width="672" />

Hmmmm. Seems not to be working. Luckily, there are some workarounds


# The 'Drop In' Solution

If you already have a pipeline in place and want to just retrofit your old code without too much trouble, this may work best for you.

The `ggsurvplot_*` family of functions produce `ggplot` objects, as you might imagine. As such, they contain a `data` list item from which plots are made. `ggsurvplot` does some hard work behind the scenes to take our `survfit` and coerce it into a rectangular data frame. We can access them like so:


```r
surv_plot <- ggsurvplot_facet(fit_facet, facet.by = "id", data = combined)

surv_plot$data |> 
  head() |> 
  kable()
```



| time| n.risk| n.event| n.censor|      surv|   std.err| upper|     lower|strata            |sex |id      |
|----:|------:|-------:|--------:|---------:|---------:|-----:|---------:|:-----------------|:---|:-------|
|    0|     90|       0|        0| 1.0000000| 0.0000000|     1| 1.0000000|sex=f, id=lung    |f   |lung    |
|    0|    361|       0|        0| 1.0000000| 0.0000000|     1| 1.0000000|sex=f, id=myeloid |f   |myeloid |
|    0|    138|       0|        0| 1.0000000| 0.0000000|     1| 1.0000000|sex=m, id=lung    |m   |lung    |
|    0|    285|       0|        0| 1.0000000| 0.0000000|     1| 1.0000000|sex=m, id=myeloid |m   |myeloid |
|    5|     90|       1|        0| 0.9888889| 0.0111734|     1| 0.9674682|sex=f, id=lung    |f   |lung    |
|   60|     89|       1|        0| 0.9777778| 0.0158910|     1| 0.9477934|sex=f, id=lung    |f   |lung    |

From this point, we just need to use the `geom_step` function from `ggplot2`:




```r
surv_plot_data <- surv_plot$data

censored <- surv_plot_data |> filter(n.censor == 1)

surv_plot_data |> 
  ggplot(aes(time, surv, color = sex)) + 
  geom_step() + 
  geom_point(data = censored, shape = 3) + 
  facet_grid(~id, scales = "free_x")
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-6-1.png" width="672" />

This won't get you 100% of the way for everything - risk tables, for instance, are not a drop in fix. But for Kaplan-Meier curves alone, this can get you fairly far! Plus, I find it's a bit easier to customize this way:


```r
surv_plot_data <- surv_plot$data |> 
  mutate(time = time/365.25)

censored <- surv_plot_data |> filter(n.censor == 1)

surv_plot_data |> 
  ggplot(aes(x = time, y = surv, color = sex)) + 
  geom_step() + 
  labs(x = "Time (years)", y = "Overall Survival", color = "Sex") +
  geom_point(data = censored, shape = 3) + 
  facet_grid(~id, scales = "free_x") +
  bladdr::theme_tufte(font_size = 15)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-7-1.png" width="672" />

# The 'survminer-less' Solution

You can get around using `survminer` all together by using `broom::tidy`:


```r
tidy_fit_data <- fit_facet |> 
  broom::tidy() |> 
  separate(strata, c("stratum1", "stratum2"), sep = ", ") |> 
  pivot_longer(starts_with("stratum")) |> 
  mutate(name = str_extract(value, "^[^=]*"), # Please don't make fun of my regex
         value = str_remove(value, paste0(name, "="))) |> 
  pivot_wider() |> 
  mutate(time = time/365.25)

tidy_censored <- tidy_fit_data |> filter(n.censor == 1)

tidy_fit_data |> 
  ggplot(aes(x = time, y = estimate, color = sex)) + 
  geom_step() + 
  labs(x = "Time (years)", y = "Overall Survival", color = "Sex") +
  geom_point(data = tidy_censored, shape = 3) + 
  facet_grid(~id, scales = "free_x") +
  bladdr::theme_tufte(font_size = 15)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-8-1.png" width="672" />


# Ok, but that sucks.

Yeah. I get it. A lot of code for what used to be just a little. You DO gain some autonomy in the nice syntax of `ggplot2`, but the initial tidying steps suck. Fortunately, there are some things you can do to make your life a little easier.

David Robinson ([@drob](https://twitter.com/drob)) has a personal package that performs the whole strata-munging mess for us:


```r
# devtools::install_github("https://github.com/dgrtwo/drlib")
library(drlib)

tidy_fit_data <- fit_facet |> 
  broom::tidy() |>
  extract_strata(strata) |> 
  mutate(time = time/365.25)

tidy_censored <- tidy_fit_data |>
  filter(n.censor == 1)

tidy_fit_data |> 
  ggplot(aes(x = time, y = estimate, color = sex)) + 
  geom_step() + 
  labs(x = "Time (years)", y = "Overall Survival", color = "Sex") +
  geom_point(data = tidy_censored, shape = 3) + 
  facet_grid(~id, scales = "free_x") +
  bladdr::theme_tufte(font_size = 15)
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-9-1.png" width="672" />

Et voilà! Two beautiful curves with minimal code (at least, on our part).

It would be even nicer if this feature was rolled in to `broom::tidy` itself (and there is a [now-stale and auto-closed issue on github](https://github.com/tidymodels/broom/issues/765) that sought to do this), but I also understand that auto-extraction of strata like this isn't without its challenges, _particularly_ with more complex formulas involving, say, `I()`, interactions, etc. Still, it would be nice to be able to have as an argument, say `.extract_strata = TRUE`.

# PS

In a future blog posts, I may go over how to replicate other features typically provided by `survminer`, like including p-values on plots.

# Thanks

The thumbnail photo was taken by <a href="https://unsplash.com/@david113?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">David Tip</a> on <a href="https://unsplash.com/s/photos/desert-island?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>

Thank you David Robinson, Alboukadel Kassambara, and the entire tidyverse team for producing excellent tools!
