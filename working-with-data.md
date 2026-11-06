---
title: "Working with data"
teaching: 90
exercises: 30
---
  
:::::::::::::::::::::::::::::::::::::: questions 

- How do you manipulate tabular data in R?

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- Import CSV data into R.
- Understand the difference between base R and `tidyverse` approaches.
- Subset rows and columns of data.frames.
- Use pipes to link steps together into pipelines.
- Create new data.frame columns using existing columns.
- Utilize the concept of split-apply-combine data analysis.
- Reshape data between wide and long formats.
- Export data to a CSV file.

::::::::::::::::::::::::::::::::::::::::::::::::
  


## Setup

:::::::::::::::::::::::::::::::::::::: callout

At the end of the last episode we modified `surveys$sex` directly, overwriting the original column.
That's a reasonable thing to do interactively, but notice that those changes exist only in our R session, the CSV on disk is unchanged.

Re-reading the file gives us back the original data.
This is actually a feature: our raw data stays raw, and any modifications live in our script, so they can be re-run and inspected.
It also means that from here on we'll stop modifying columns in place and instead build pipelines that take the data in and produce a modified copy out.

::::::::::::::::::::::::::::::::::::::::::::::

Let's start clean for this episode, in a new script with an empty environment.
As if we had closed R/RStudio and we just opened it again.
We can take our now usual process of, first, loading the required libraries, and second, reading our data.


``` r
library(tidyverse)

surveys <- read_csv("data/cleaned/surveys_complete_77_89.csv")
```

### Our research question

In the last episode we asked you to note down analyses you wanted to do but didn't know how.
Many of those probably sound like: "I wanted the mean weight for each species," or "I wanted to look at only the animals caught in a particular plot," or "I wanted to compare 1977 to 1989."
Every one of those involves subsetting, grouping, or creating new variables, and that's what this episode is about.

There's one question in particular that we left unanswered. When we plotted the distribution of weights, we saw something odd:


``` r
ggplot(data = surveys, aes(x = weight)) +
  geom_histogram()
```

``` output
`stat_bin()` using `bins = 30`. Pick better value `binwidth`.
```

<img src="fig/working-with-data-rendered-hist-1.png" alt="Histogram of the weight of our samples" width="600" height="600" style="display: block; margin: auto;" />

This distribution has two humps: a big cluster of light animals under about 50 grams, and a second, smaller cluster somewhere around 90 to 150 grams.
A distribution with two humps is usually a hint that we are not looking at one thing, but at **two or more different things mixed together**.
So our research question for this episode is:

> Why is the distribution of rodent weights bimodal?

Our leading hypothesis is the obvious one for anyone who has looked at a rodent: the dataset contains multiple species, and different species are different sizes.
Smaller species make up the first hump, larger species the second.

### Manipulating data

The trouble is that none of the tools we have learned so far can test this. 
`summary()` and `quantile()` describe the whole dataset at once.
To explain structure within a dataset, we need to be able to split it into pieces, compute something for each piece, and compare the results.
That is exactly what the `dplyr` and `tidyr` packages let us do.

Here's how we'll proceed:

1. Narrow the data to the rows and columns our question needs (`select()`, `filter()`, and the pipe -`%>%` or `|>`-).
2. Test the hypothesis by calculating weights per species (`group_by()` and `summarize()`).
3. Look at what's left over once species is accounted for, which is where sex and individual variation live (`group_by()` with `mutate()`).
4. Check whether body size changes over time (`mutate()` and working with dates with the `lubridate` package).
5. Build a shareable summary table and export it (`pivot_wider()` and `write_csv()`).

By the end we should have an answer, a plot, and a CSV file we could hand to a collaborator.

## Selecting and filtering

Our question is about weight and species.
The surveys data.frame has 13 columns, most of which are irrelevant right now, and it has rows where weight was never recorded.
Before we can investigate anything, we need to cut the data down to what is actually usable.

After EDA, this is almost always the first step of a real analysis, and `dplyr` gives us two functions for it: `select()`, which picks columns, and `filter()`, which picks rows.

:::::::::::::::::::::::::::::::::::::::::: callout

Between `select()` and `filter()`, it can be hard to remember which operates on columns and which operates on rows.
`sele`**`c`**`t()` has a **c** for **c**olumns and `filte`**`r`**`()` has an **r** for **r**ows.

::::::::::::::::::::::::::::::::::::::::::

### `select()`: choosing columns

To use the `select()` function, the first argument is the name of the data.frame, and the rest of the arguments are the names of the columns you want.
For our question, the columns that matter are the identity of the animal and its measurements:



``` r
select(surveys, species_id, sex, plot_id, weight, hindfoot_length)
```

``` output
# A tibble: 16,878 × 5
   species_id sex   plot_id weight hindfoot_length
   <chr>      <chr>   <dbl>  <dbl>           <dbl>
 1 NL         M           2     NA              32
 2 NL         M           3     NA              33
 3 DM         F           2     NA              37
 4 DM         M           7     NA              36
 5 DM         M           3     NA              35
 6 PF         M           1     NA              14
 7 PE         F           2     NA              NA
 8 DM         M           1     NA              37
 9 DM         F           1     NA              34
10 PF         F           6     NA              20
# ℹ 16,868 more rows
```

The columns are arranged in the order we specified inside the function, which is useful for putting the most important variables first.

Sometimes it's easier to say what you don't want. To drop specific columns, put a `-` in front of them:


``` r
select(surveys, -record_id, -genus, -species)
```

``` output
# A tibble: 16,878 × 10
   month   day  year plot_id species_id sex   hindfoot_length weight taxa  
   <dbl> <dbl> <dbl>   <dbl> <chr>      <chr>           <dbl>  <dbl> <chr> 
 1     7    16  1977       2 NL         M                  32     NA Rodent
 2     7    16  1977       3 NL         M                  33     NA Rodent
 3     7    16  1977       2 DM         F                  37     NA Rodent
 4     7    16  1977       7 DM         M                  36     NA Rodent
 5     7    16  1977       3 DM         M                  35     NA Rodent
 6     7    16  1977       1 PF         M                  14     NA Rodent
 7     7    16  1977       2 PE         F                  NA     NA Rodent
 8     7    16  1977       1 DM         M                  37     NA Rodent
 9     7    16  1977       1 DM         F                  34     NA Rodent
10     7    16  1977       6 PF         F                  20     NA Rodent
# ℹ 16,868 more rows
# ℹ 1 more variable: plot_type <chr>
```

We can also select columns based on a property rather than by name, using the `where()` function. A useful question at this stage of an analysis is "which of my variables could I even take a mean of?" Those are the numeric ones:


``` r
select(surveys, where(is.numeric))
```

``` output
# A tibble: 16,878 × 7
   record_id month   day  year plot_id hindfoot_length weight
       <dbl> <dbl> <dbl> <dbl>   <dbl>           <dbl>  <dbl>
 1         1     7    16  1977       2              32     NA
 2         2     7    16  1977       3              33     NA
 3         3     7    16  1977       2              37     NA
 4         4     7    16  1977       7              36     NA
 5         5     7    16  1977       3              35     NA
 6         6     7    16  1977       1              14     NA
 7         7     7    16  1977       2              NA     NA
 8         8     7    16  1977       1              37     NA
 9         9     7    16  1977       1              34     NA
10        10     7    16  1977       6              20     NA
# ℹ 16,868 more rows
```

:::::::::::::::::::::::::::::::::::::::::: spoiler

### More about select

`select()` is more flexible than we have space to show. A few other things it accepts:

- Column positions: `select(surveys, c(3:5, 10))` gives you the 3rd, 4th, 5th, and 10th columns.
  Be careful with this, since you are being less explicit about what you want, and the result will change if the column order changes.
- Other helper functions like `contains()`, `starts_with()`, and `ends_with()` for matching column names, and `where(anyNA)` to find columns containing missing values.

Running ?dplyr_tidy_select will show you the full list.

::::::::::::::::::::::::::::::::::::::::::


### `filter()`: choosing rows

The `filter()` function keeps rows that meet a condition.
The condition we most need right now is that weight was actually recorded.
The `is.na()` function asks "is this value missing?", and putting ! in front of something means "not", so `!is.na(weight)` means "weight is not missing":


``` r
filter(surveys, !is.na(weight))
```

``` output
# A tibble: 15,186 × 13
   record_id month   day  year plot_id species_id sex   hindfoot_length weight
       <dbl> <dbl> <dbl> <dbl>   <dbl> <chr>      <chr>           <dbl>  <dbl>
 1        63     8    19  1977       3 DM         M                  35     40
 2        64     8    19  1977       7 DM         M                  37     48
 3        65     8    19  1977       4 DM         F                  34     29
 4        66     8    19  1977       4 DM         F                  35     46
 5        67     8    19  1977       7 DM         M                  35     36
 6        68     8    19  1977       8 DO         F                  32     52
 7        69     8    19  1977       2 PF         M                  15      8
 8        70     8    19  1977       3 OX         F                  21     22
 9        71     8    19  1977       7 DM         F                  36     35
10        74     8    19  1977       8 PF         M                  12      7
# ℹ 15,176 more rows
# ℹ 4 more variables: genus <chr>, species <chr>, taxa <chr>, plot_type <chr>
```

We've gone **from 16878 rows to 15186 rows**.
Rather than handling them with `na.rm = TRUE` every time we do a calculation, we are now dealing with them structurally: we remove them once, at the start, and then everything downstream is clean.

This function works with all the comparison operators.
To get all the rows where year is equal to 1985:


``` r
filter(surveys, year == 1985)
```

``` output
# A tibble: 1,438 × 13
   record_id month   day  year plot_id species_id sex   hindfoot_length weight
       <dbl> <dbl> <dbl> <dbl>   <dbl> <chr>      <chr>           <dbl>  <dbl>
 1      9790     1    19  1985      16 RM         F                  16      4
 2      9791     1    19  1985      17 OT         F                  20     16
 3      9792     1    19  1985       6 DO         M                  35     48
 4      9793     1    19  1985      12 DO         F                  35     40
 5      9794     1    19  1985      24 RM         M                  16      4
 6      9795     1    19  1985      12 DO         M                  34     48
 7      9796     1    19  1985       6 DM         F                  37     35
 8      9797     1    19  1985      14 DM         M                  36     45
 9      9798     1    19  1985       6 DM         F                  36     38
10      9799     1    19  1985      19 RM         M                  16      4
# ℹ 1,428 more rows
# ℹ 4 more variables: genus <chr>, species <chr>, taxa <chr>, plot_type <chr>
```

The `==` sign means "is equal to".
There are several other operators we can use: `>`, `>=`, `<`, `<=`, and `!=` (not equal to).
Another useful operator is `%in%`, which asks if the value on the left-hand side is found anywhere in the vector on the right-hand side.
For example, to pull out two particular species:


``` r
filter(surveys, species_id %in% c("RM", "DO"))
```

``` output
# A tibble: 2,835 × 13
   record_id month   day  year plot_id species_id sex   hindfoot_length weight
       <dbl> <dbl> <dbl> <dbl>   <dbl> <chr>      <chr>           <dbl>  <dbl>
 1        68     8    19  1977       8 DO         F                  32     52
 2       292    10    17  1977       3 DO         F                  36     33
 3       294    10    17  1977       3 DO         F                  37     50
 4       311    10    17  1977      19 RM         M                  18     13
 5       317    10    17  1977      17 DO         F                  32     48
 6       323    10    17  1977      17 DO         F                  33     31
 7       337    10    18  1977       8 DO         F                  35     41
 8       356    11    12  1977       1 DO         F                  32     44
 9       378    11    12  1977       1 DO         M                  33     48
10       397    11    13  1977      17 RM         F                  16      7
# ℹ 2,825 more rows
# ℹ 4 more variables: genus <chr>, species <chr>, taxa <chr>, plot_type <chr>
```

We can also combine conditions in one filter() call, using `&` for "and" and `|` for "or".
Here we get rows from 1988 or earlier that have a recorded hindfoot length:


``` r
filter(surveys, year <= 1988 & !is.na(hindfoot_length))
```

``` output
# A tibble: 12,779 × 13
   record_id month   day  year plot_id species_id sex   hindfoot_length weight
       <dbl> <dbl> <dbl> <dbl>   <dbl> <chr>      <chr>           <dbl>  <dbl>
 1         1     7    16  1977       2 NL         M                  32     NA
 2         2     7    16  1977       3 NL         M                  33     NA
 3         3     7    16  1977       2 DM         F                  37     NA
 4         4     7    16  1977       7 DM         M                  36     NA
 5         5     7    16  1977       3 DM         M                  35     NA
 6         6     7    16  1977       1 PF         M                  14     NA
 7         8     7    16  1977       1 DM         M                  37     NA
 8         9     7    16  1977       1 DM         F                  34     NA
 9        10     7    16  1977       6 PF         F                  20     NA
10        11     7    16  1977       5 DS         F                  53     NA
# ℹ 12,769 more rows
# ℹ 4 more variables: genus <chr>, species <chr>, taxa <chr>, plot_type <chr>
```

::::::::::::::::::::::::::::::::::::: challenge

### Challenge 1: Narrowing down to a subset of interest
The genus Dipodomys (kangaroo rats) contains some of the larger animals in this dataset, so it's a good candidate for the second hump of our histogram.
The species IDs for this genus are DM, DO, and DS.

Make a data.frame called `dipodomys` containing only the Dipodomys records that have a recorded weight.

:::::::::::::::::::::::: solution


``` r
dipodomys <- filter(surveys, species_id %in% c("DM", "DO", "DS") & !is.na(weight))
```

You could also write this with three `|` conditions, but `%in%` is much more compact:


``` r
dipodomys <- filter(surveys, 
                    (species_id == "DM" | species_id == "DO" | species_id == "DS") & 
                      !is.na(weight))
```

Note the parentheses in the second version. Without them, the & would only apply to the last condition.

::::::::::::::::::::::::

What is the mean weight of these animals?
How does it compare to the mean weight of the whole dataset (which was 53.22 grams)?

:::::::::::::::::::::::: solution

``` r
mean(dipodomys$weight)
```

``` output
[1] 62.67743
```

Considerably heavier than the dataset as a whole, which is consistent with our hypothesis that these larger species are responsible for the upper hump of the distribution.
But this is only three species out of many, and we had to know their IDs in advance.
We need a way to do this for all species at once.

::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::


## The pipe `%>%` or `|>`

Notice that in the challenge above, we needed to do two things at once:
pick certain rows and ignore missing weights.
And really we should also be dropping the columns we don't need.
What happens when we want to combine `select()` and `filter()`?

We have a couple of options.
First, we could use nested functions:


``` r
filter(select(surveys, -record_id, -genus, -species), !is.na(weight))
```

``` output
# A tibble: 15,186 × 10
   month   day  year plot_id species_id sex   hindfoot_length weight taxa  
   <dbl> <dbl> <dbl>   <dbl> <chr>      <chr>           <dbl>  <dbl> <chr> 
 1     8    19  1977       3 DM         M                  35     40 Rodent
 2     8    19  1977       7 DM         M                  37     48 Rodent
 3     8    19  1977       4 DM         F                  34     29 Rodent
 4     8    19  1977       4 DM         F                  35     46 Rodent
 5     8    19  1977       7 DM         M                  35     36 Rodent
 6     8    19  1977       8 DO         F                  32     52 Rodent
 7     8    19  1977       2 PF         M                  15      8 Rodent
 8     8    19  1977       3 OX         F                  21     22 Rodent
 9     8    19  1977       7 DM         F                  36     35 Rodent
10     8    19  1977       8 PF         M                  12      7 Rodent
# ℹ 15,176 more rows
# ℹ 1 more variable: plot_type <chr>
```

This is already hard to read with two functions, and it becomes nearly impossible with five.

An alternative approach is to create intermediate objects:


``` r
surveys_sub <- select(surveys, -record_id, -genus, -species)
surveys_weight <- filter(surveys_sub, !is.na(weight))
```

This is easier to read, since the steps are in order, but after enough steps we are left with a cluttered environment full of near-identical objects with confusing names.

An elegant solution is an operator called the pipe, which comes in two variations: `%>%` and `|>`.
They both work the same and are replacable the one with the other in the latest versions of R.
You can insert it with the keyboard shortcut <kbd>Shift+Cmd+M</kbd> (Mac) or <kbd>Shift+Ctrl+M</kbd> (Windows):


``` r
surveys %>% 
  select(-record_id, -genus, -species) %>% 
  filter(!is.na(weight))
```

``` output
# A tibble: 15,186 × 10
   month   day  year plot_id species_id sex   hindfoot_length weight taxa  
   <dbl> <dbl> <dbl>   <dbl> <chr>      <chr>           <dbl>  <dbl> <chr> 
 1     8    19  1977       3 DM         M                  35     40 Rodent
 2     8    19  1977       7 DM         M                  37     48 Rodent
 3     8    19  1977       4 DM         F                  34     29 Rodent
 4     8    19  1977       4 DM         F                  35     46 Rodent
 5     8    19  1977       7 DM         M                  35     36 Rodent
 6     8    19  1977       8 DO         F                  32     52 Rodent
 7     8    19  1977       2 PF         M                  15      8 Rodent
 8     8    19  1977       3 OX         F                  21     22 Rodent
 9     8    19  1977       7 DM         F                  36     35 Rodent
10     8    19  1977       8 PF         M                  12      7 Rodent
# ℹ 15,176 more rows
# ℹ 1 more variable: plot_type <chr>
```

The pipe takes the thing on the left-hand side and inserts it as the first argument of the function on the right-hand side.
By putting each function on a new line, we build a readable pipeline.
It can be useful to think of this as an assembly line for our data:
it starts at the top, gets piped into `select()` and comes out modified, then gets sent into `filter()` where it is modified further, and the final product prints to the console.

It can also be helpful to read the pipe as meaning "and then".
Since many tidyverse functions are named with verbs, a pipeline reads almost like a sentence:
take surveys, and then select these columns, and then filter these rows.

:::::::::::::::::::::::::::::::::::::::::::: instructor
It's worth showing the learners that you can run a pipeline without highlighting the whole thing. If your cursor is on any line of a pipeline, running that line will run the whole thing.

You can also show that by highlighting a section of a pipeline, you can run only the first X steps of it. This is the single most useful debugging habit for pipelines, and it's worth demonstrating deliberately.
::::::::::::::::::::::::::::::::::::::::::::

Now we can build the subset we actually want for our analysis.
We need `species_id` (to group by), `sex` and `plot_id` (for later), and `weight` and `hindfoot_length` (our measurements).
And we need rows with a weight value:


``` r
surveys_weight <- surveys %>% 
  filter(!is.na(weight)) %>% 
  select(species_id, sex, plot_id, year, month, day, weight, hindfoot_length)

surveys_weight
```

``` output
# A tibble: 15,186 × 8
   species_id sex   plot_id  year month   day weight hindfoot_length
   <chr>      <chr>   <dbl> <dbl> <dbl> <dbl>  <dbl>           <dbl>
 1 DM         M           3  1977     8    19     40              35
 2 DM         M           7  1977     8    19     48              37
 3 DM         F           4  1977     8    19     29              34
 4 DM         F           4  1977     8    19     46              35
 5 DM         M           7  1977     8    19     36              35
 6 DO         F           8  1977     8    19     52              32
 7 PF         M           2  1977     8    19      8              15
 8 OX         F           3  1977     8    19     22              21
 9 DM         F           7  1977     8    19     35              36
10 PF         M           8  1977     8    19      7              12
# ℹ 15,176 more rows
```

This `surveys_weight` object is the dataset we will use for the rest of the episode.
Notice that we did not overwrite surveys.
If we make a mistake, or realize we need a column we dropped, the full data is still sitting there untouched in the R environment.

::::::::::::::::::::::::::::::::::::: challenge

### Challenge 2: Using pipes
Use the surveys data to make a data.frame that has the columns `species_id`, `sex`, and `weight`, containing only data from the year 1988 where `sex` was recorded.

:::::::::::::::::::::::: solution


``` r
surveys_1988 <- surveys %>%
  filter(year == 1988 & !is.na(sex)) %>%
  select(species_id, sex, weight)
```

Make sure to `filter()` before you `select()`.
You need the year column for filtering rows, but it is discarded in the `select()` step, so doing it the other way around gives you an error:


``` r
surveys %>%
  select(species_id, sex, weight) %>% 
  filter(year == 1988 & !is.na(sex))
```

``` error
Error in `filter()`:
ℹ In argument: `year == 1988 & !is.na(sex)`.
Caused by error in `year == 1988`:
! comparison (==) is possible only for atomic and list types
```

This is a very common error, and the message is a useful one to learn to recognize:
R is telling you that the column you asked about doesn't exist, because you removed it a step earlier.

Also make sure to use `==` instead of `=` when filtering for rows where year is equal to 1988.

::::::::::::::::::::::::
::::::::::::::::::::::::::::::::::::::::::::::::


## Grouping: The split-apply-combine approach

Now we can get to the actual question.
Our hypothesis is that the two humps in the weight distribution correspond to different species.
To test it, we need the mean weight of each species so we can see whether they fall into distinct groups.

This is an instance of a very general pattern called the split-apply-combine approach:
you split the data into groups, apply some analysis to each group, and combine the results back together into a single table.
An enormous proportion of data analysis tasks fit this pattern, and `dplyr` has two functions that implement it:`group_by()` and `summarize()`.

`group_by()` takes a data.frame and the names of one or more columns containing categorical values that define the groups. `summarize()` then collapses each group down to a one-row summary, giving us back a data.frame with one row per group.

:::::::::::::::::::::::::::::::::::::::::: spoiler

summary() vs. summarize()

These two names are confusingly similar, and they do quite different things.

- `summary()` is a base R function. You hand it a data.frame and it decides what to show you: for every column, a fixed set of statistics (min, median, mean, quartiles, max, NA count for numeric columns).
-` summarize()` is a `dplyr` function. You decide what to calculate, you name the resulting columns, and crucially it can operate per group when combined with `group_by()`.

`summary()` is for a quick look at everything. `summarize()` is for computing specific things you've decided you want.
You can also spell it `summarise()` if you prefer British English; `dplyr` accepts both.

::::::::::::::::::::::::::::::::::::::::::

Let's calculate the mean weight of each species:


``` r
surveys_weight %>% 
  group_by(species_id) %>% 
  summarize(mean_weight = mean(weight))
```

``` output
# A tibble: 18 × 2
   species_id mean_weight
   <chr>            <dbl>
 1 BA                7   
 2 DM               42.5 
 3 DO               48.9 
 4 DS              120.  
 5 NL              158.  
 6 OL               32.6 
 7 OT               23.8 
 8 OX               21   
 9 PE               21.7 
10 PF                7.09
11 PH               30.7 
12 PM               21.6 
13 PP               16.3 
14 RF               13.4 
15 RM               10.3 
16 SF               54.8 
17 SH               75.0 
18 SS               93.5 
```

There is our answer beginning to appear.
The mean weights are not spread evenly, there are species averaging around 8 grams, others around 40 or 50, and some well over 100.
Let's make this easier to read by sorting it, using the `arrange()` function:


``` r
surveys_weight %>% 
  group_by(species_id) %>% 
  summarize(mean_weight = mean(weight)) %>% 
  arrange(mean_weight)
```

``` output
# A tibble: 18 × 2
   species_id mean_weight
   <chr>            <dbl>
 1 BA                7   
 2 PF                7.09
 3 RM               10.3 
 4 RF               13.4 
 5 PP               16.3 
 6 OX               21   
 7 PM               21.6 
 8 PE               21.7 
 9 OT               23.8 
10 PH               30.7 
11 OL               32.6 
12 DM               42.5 
13 DO               48.9 
14 SF               54.8 
15 SH               75.0 
16 SS               93.5 
17 DS              120.  
18 NL              158.  
```

To reverse the order, wrap the column name in `desc()`:


``` r
surveys_weight %>% 
  group_by(species_id) %>% 
  summarize(mean_weight = mean(weight)) %>% 
  arrange(desc(mean_weight))
```

``` output
# A tibble: 18 × 2
   species_id mean_weight
   <chr>            <dbl>
 1 NL              158.  
 2 DS              120.  
 3 SS               93.5 
 4 SH               75.0 
 5 SF               54.8 
 6 DO               48.9 
 7 DM               42.5 
 8 OL               32.6 
 9 PH               30.7 
10 OT               23.8 
11 PE               21.7 
12 PM               21.6 
13 OX               21   
14 PP               16.3 
15 RF               13.4 
16 RM               10.3 
17 PF                7.09
18 BA                7   
```

Now the structure is clear.
A handful of species (NL, DS, SH, SF) have mean weights well above 70 grams, while the great majority sit below 50.
But a mean on its own can be misleading, a species averaging 100 grams tells us very little if we only ever caught two of them.
We should also count how many observations are in each group.
The function `n()` does exactly that, and `summarize()` lets us define several columns in one call:


``` r
surveys_weight %>% 
  group_by(species_id) %>% 
  summarize(mean_weight = mean(weight),
            n = n()) %>% 
  arrange(desc(mean_weight))
```

``` output
# A tibble: 18 × 3
   species_id mean_weight     n
   <chr>            <dbl> <int>
 1 NL              158.     824
 2 DS              120.    2255
 3 SS               93.5      2
 4 SH               75.0     95
 5 SF               54.8      5
 6 DO               48.9   1428
 7 DM               42.5   5481
 8 OL               32.6    758
 9 PH               30.7     13
10 OT               23.8    822
11 PE               21.7    828
12 PM               21.6    409
13 OX               21        6
14 PP               16.3    368
15 RF               13.4     62
16 RM               10.3   1314
17 PF                7.09   513
18 BA                7        3
```

This is the answer to our research question.
DS has a mean weight of about 120 grams with over 2000 observations, and NL averages around 158 grams with nearly 800 observations.
These are not rare flukes, they are genuinely abundant large species.
Meanwhile DM averages about 42 grams with over 5000 observations, making it the single biggest contributor to the lower hump.

The bimodality is real, and it is a species effect.
The dataset is dominated by a very large number of small-bodied individuals (mostly DM, PP, PF, RM) plus a substantial number of large-bodied ones (DS, NL), with comparatively few species in between.
Mixing those together produces two humps.

We can confirm this visually by coloring the original histogram by species:


``` r
surveys_weight %>% 
  ggplot(aes(x = weight, fill = species_id)) +
  geom_histogram()
```

``` output
`stat_bin()` using `bins = 30`. Pick better value `binwidth`.
```

<img src="fig/working-with-data-rendered-hist-species-1.png" alt="Histogram of the weight of our samples, colored by species_id" width="600" height="600" style="display: block; margin: auto;" />

The legend is unwieldy with this many species, but the point comes across: the two humps are composed of different species.

### Grouping by more than one variable
You will often want to form groups from combinations of columns.
For instance, we might wonder whether the sexes differ in weight within each species.
All we have to do is add another column to group_by():


``` r
surveys_weight %>% 
  group_by(species_id, sex) %>% 
  summarize(mean_weight = mean(weight),
            n = n())
```

``` output
`summarise()` has regrouped the output.
ℹ Summaries were computed grouped by species_id and sex.
ℹ Output is grouped by species_id.
ℹ Use `summarise(.groups = "drop_last")` to silence this message.
ℹ Use `summarise(.by = c(species_id, sex))` for per-operation grouping
  (`?dplyr::dplyr_by`) instead.
```

``` output
# A tibble: 46 × 4
# Groups:   species_id [18]
   species_id sex   mean_weight     n
   <chr>      <chr>       <dbl> <int>
 1 BA         M             7       3
 2 DM         F            40.7  2460
 3 DM         M            44.0  3013
 4 DM         <NA>         37       8
 5 DO         F            48.4   679
 6 DO         M            49.3   748
 7 DO         <NA>         44       1
 8 DS         F           118.   1055
 9 DS         M           123.   1184
10 DS         <NA>        121.     16
# ℹ 36 more rows
```

::::::::::::::::::::::::::::::::::::::::::::::: spoiler

### A note on grouped output

If you look at the output printed in your console, you will see lines like:

``` output
# A tibble: 46 × 4
# Groups:   species_id [18]
```

That tells us we have 46 rows, 4 columns, and a grouping by `species_id` with 18 groups. We see something similar if we use `group_by()` on its own:


``` r
surveys_weight %>% 
  group_by(species_id, sex)
```

``` output
# A tibble: 15,186 × 8
# Groups:   species_id, sex [46]
   species_id sex   plot_id  year month   day weight hindfoot_length
   <chr>      <chr>   <dbl> <dbl> <dbl> <dbl>  <dbl>           <dbl>
 1 DM         M           3  1977     8    19     40              35
 2 DM         M           7  1977     8    19     48              37
 3 DM         F           4  1977     8    19     29              34
 4 DM         F           4  1977     8    19     46              35
 5 DM         M           7  1977     8    19     36              35
 6 DO         F           8  1977     8    19     52              32
 7 PF         M           2  1977     8    19      8              15
 8 OX         F           3  1977     8    19     22              21
 9 DM         F           7  1977     8    19     35              36
10 PF         M           8  1977     8    19      7              12
# ℹ 15,176 more rows
```

We get back the whole data.frame, apparently unchanged, but with grouping information attached.
`group_by()` doesn't change the data, it changes how later functions treat the data.

Groups persist through a pipeline, and if you assign the result to an object, that object stays grouped.
This can produce baffling results later on when you have forgotten the grouping is there.
It is therefore a good habit to call ungroup() at the end of any pipeline containing group_by():


``` r
species_weights <- surveys_weight %>% 
  group_by(species_id) %>% 
  summarize(mean_weight = mean(weight),
            n = n()) %>% 
  arrange(desc(mean_weight)) %>% 
  ungroup()

species_weights
```

``` output
# A tibble: 18 × 3
   species_id mean_weight     n
   <chr>            <dbl> <int>
 1 NL              158.     824
 2 DS              120.    2255
 3 SS               93.5      2
 4 SH               75.0     95
 5 SF               54.8      5
 6 DO               48.9   1428
 7 DM               42.5   5481
 8 OL               32.6    758
 9 PH               30.7     13
10 OT               23.8    822
11 PE               21.7    828
12 PM               21.6    409
13 OX               21        6
14 PP               16.3    368
15 RF               13.4     62
16 RM               10.3   1314
17 PF                7.09   513
18 BA                7        3
```

Now the output just says `# A tibble: 21 × 3` at the top, with no groups.

:::::::::::::::::::::::::::::::::::::::::::::::::::::::

Our result is much longer now, because we have one row per species-and-sex combination.
Many of the differences between the sexes are small, but notice that there are rows where sex is NA. Those are animals whose sex was never recorded.

:::::::::::::::::::::::::::::::::::::::::: callout

### Why we filtered NA weights early

Before we removed rows with missing weights at the very beginning of the pipeline.
Here is why that mattered.

Had we not done so, we would have had to write `mean(weight, na.rm = TRUE)` here, and for any group where every weight was missing, `na.rm = TRUE` would have removed all the values before calculating the mean.
The mean of nothing is NaN ("Not a Number"), and our results table would have been peppered with them.

By filtering once at the start, those groups simply never form, and every number in our output is a real number.
In general, dealing with missing data structurally and early produces cleaner results than patching it at every calculation.

::::::::::::::::::::::::::::::::::::::::::::::::::::


::::::::::::::::::::::::::::::::::::: challenge

### Challenge 3: Extending the summary

Last episode we calculated quantiles of weight across the entire dataset using `quantile()`.
Now we can do it per species.
Add columns to the species summary giving the minimum weight, the 10th percentile, the 90th percentile, and the maximum, for each species.

**Hint:** `quantile(weight, probs = 0.1)` gives you a single quantile.

:::::::::::::::::::::::: solution


``` r
surveys_weight %>% 
  group_by(species_id) %>% 
  summarize(min = min(weight),
            q10 = quantile(weight, probs = 0.1),
            q90 = quantile(weight, probs = 0.9),
            max = max(weight)) %>% 
  ungroup()
```

``` output
# A tibble: 18 × 5
   species_id   min   q10   q90   max
   <chr>      <dbl> <dbl> <dbl> <dbl>
 1 BA             6   6.2   7.8     8
 2 DM            13  34    50      66
 3 DO            12  39    57      73
 4 DS            12  87   146     190
 5 NL            30  99   215.    278
 6 OL            11  24    41      56
 7 OT             5  18.1  29      45
 8 OX            18  19    23      24
 9 PE             8  17    27      40
10 PF             4   6     8      15
11 PH            23  25.6  34      39
12 PM             7  14    27      35
13 PP             7  12    21      28
14 RF             9  11    16      18
15 RM             4   8    13      20
16 SF            46  46.4  69.2    82
17 SH            16  42.4 106.    140
18 SS            57  64.3 123.    130
```

This is the same calculation we did last episode, but now split by species.
Notice how much more informative it is:
the whole-dataset range of 4 to 278 grams was really describing several different animals at once.

::::::::::::::::::::::::



What do you think would happen if you ran `group_by(weight)` instead of `group_by(species_id)`?
Try it with `summarize(n = n()`, and explain the result.

:::::::::::::::::::::::: solution


``` r
surveys_weight %>% 
  group_by(weight) %>% 
  summarize(n = n())
```

``` output
# A tibble: 252 × 2
   weight     n
    <dbl> <int>
 1      4    15
 2      5    45
 3      6   132
 4      7   254
 5      8   247
 6      9   265
 7     10   340
 8     11   307
 9     12   188
10     13   135
# ℹ 242 more rows
```

`group_by()` will happily group by a continuous variable, but it creates one group for every distinct value, which here gives us a count of how many animals weighed exactly 10 grams, exactly 11 grams, and so on.
That is occasionally useful, but it is not a summary in any meaningful sense.
`group_by()` is designed for categorical variables, and it is up to you to supply a sensible one.

::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::


## Making new columns with `mutate()`

We have explained the bimodality: it comes from mixing species of very different sizes.
But that raises a follow-up question.
Within a single species, animals still vary, some `DM` individuals are heavier than others.
How much variation is there within species, and can we find the unusually large and unusually small individuals?

To answer this we need something slightly different.
We don't want one row per species anymore, we want to keep every individual animal, but attach to each one the mean weight of its species so we can compare.
For that, we combine `group_by()` with `mutate()` instead of `summarize()`.

### What is `mutate()`

Before we do that, let's introduce `mutate()` properly.
It creates new columns based on existing ones.
For instance, our weights are in grams, but suppose a collaborator works in kilograms:


``` r
surveys_weight %>% 
  mutate(weight_kg = weight / 1000)
```

``` output
# A tibble: 15,186 × 9
   species_id sex   plot_id  year month   day weight hindfoot_length weight_kg
   <chr>      <chr>   <dbl> <dbl> <dbl> <dbl>  <dbl>           <dbl>     <dbl>
 1 DM         M           3  1977     8    19     40              35     0.04 
 2 DM         M           7  1977     8    19     48              37     0.048
 3 DM         F           4  1977     8    19     29              34     0.029
 4 DM         F           4  1977     8    19     46              35     0.046
 5 DM         M           7  1977     8    19     36              35     0.036
 6 DO         F           8  1977     8    19     52              32     0.052
 7 PF         M           2  1977     8    19      8              15     0.008
 8 OX         F           3  1977     8    19     22              21     0.022
 9 DM         F           7  1977     8    19     35              36     0.035
10 PF         M           8  1977     8    19      7              12     0.007
# ℹ 15,176 more rows
```

You can create multiple columns in one `mutate()` call, and they are created in the order you write them, which means you can refer to a new column while creating the next one:


``` r
surveys_weight %>% 
  mutate(weight_kg = weight / 1000,
         weight_lbs = weight_kg * 2.2)
```

``` output
# A tibble: 15,186 × 10
   species_id sex   plot_id  year month   day weight hindfoot_length weight_kg
   <chr>      <chr>   <dbl> <dbl> <dbl> <dbl>  <dbl>           <dbl>     <dbl>
 1 DM         M           3  1977     8    19     40              35     0.04 
 2 DM         M           7  1977     8    19     48              37     0.048
 3 DM         F           4  1977     8    19     29              34     0.029
 4 DM         F           4  1977     8    19     46              35     0.046
 5 DM         M           7  1977     8    19     36              35     0.036
 6 DO         F           8  1977     8    19     52              32     0.052
 7 PF         M           2  1977     8    19      8              15     0.008
 8 OX         F           3  1977     8    19     22              21     0.022
 9 DM         F           7  1977     8    19     35              36     0.035
10 PF         M           8  1977     8    19      7              12     0.007
# ℹ 15,176 more rows
# ℹ 1 more variable: weight_lbs <dbl>
```

### `group_by()` with `mutate()`

Now for the within-species comparison.
If we `group_by(species_id)` and then `mutate()`, the calculation is performed per group, but the result is written back onto every row of that group:


``` r
surveys_weight %>% 
  group_by(species_id) %>% 
  mutate(mean_weight = mean(weight),
         weight_diff = weight - mean_weight)
```

``` output
# A tibble: 15,186 × 10
# Groups:   species_id [18]
   species_id sex   plot_id  year month   day weight hindfoot_length mean_weight
   <chr>      <chr>   <dbl> <dbl> <dbl> <dbl>  <dbl>           <dbl>       <dbl>
 1 DM         M           3  1977     8    19     40              35       42.5 
 2 DM         M           7  1977     8    19     48              37       42.5 
 3 DM         F           4  1977     8    19     29              34       42.5 
 4 DM         F           4  1977     8    19     46              35       42.5 
 5 DM         M           7  1977     8    19     36              35       42.5 
 6 DO         F           8  1977     8    19     52              32       48.9 
 7 PF         M           2  1977     8    19      8              15        7.09
 8 OX         F           3  1977     8    19     22              21       21   
 9 DM         F           7  1977     8    19     35              36       42.5 
10 PF         M           8  1977     8    19      7              12        7.09
# ℹ 15,176 more rows
# ℹ 1 more variable: weight_diff <dbl>
```

Our new columns are at the far right and don't print, so let's use `select()` to look at just the columns of interest.
Inside `select()` we can use the `contains()` helper to grab every column with "weight" in its name:


``` r
surveys_centered <- surveys_weight %>% 
  group_by(species_id) %>% 
  mutate(mean_weight = mean(weight),
         weight_diff = weight - mean_weight) %>%
  select(species_id, sex, contains("weight"))

surveys_centered
```

``` output
# A tibble: 15,186 × 5
# Groups:   species_id [18]
   species_id sex   weight mean_weight weight_diff
   <chr>      <chr>  <dbl>       <dbl>       <dbl>
 1 DM         M         40       42.5      -2.52  
 2 DM         M         48       42.5       5.48  
 3 DM         F         29       42.5     -13.5   
 4 DM         F         46       42.5       3.48  
 5 DM         M         36       42.5      -6.52  
 6 DO         F         52       48.9       3.13  
 7 PF         M          8        7.09      0.908 
 8 OX         F         22       21         1     
 9 DM         F         35       42.5      -7.52  
10 PF         M          7        7.09     -0.0916
# ℹ 15,176 more rows
```

This is the key difference between the two approaches:

- `group_by()` + `summarize()` gives you one row per group. Use it when you want a summary table.
- `group_by()` + `mutate()` gives you all your original rows, with a per-group value added to each.
  Use it when you want to compare each observation to its group.
  
For every DM individual, the `mean_weight` column shows the same number, the mean for DM.
The `weight_diff` column then tells us how far above or below its species average each animal is.
This is often called centering, and it's a very common operation.

Now we can do something we couldn't before:
plot the within-species variation for all species on the same axis, because we have removed the between-species differences.


``` r
surveys_centered %>% 
  ggplot(aes(x = weight_diff)) +
  geom_histogram()
```

``` output
`stat_bin()` using `bins = 30`. Pick better value `binwidth`.
```

<img src="fig/working-with-data-rendered-unnamed-chunk-30-1.png" alt="" width="600" height="600" style="display: block; margin: auto;" />

The two humps are gone. Once we subtract each species' own mean, what remains is a single distribution centered on zero.
This is strong confirmation of our answer:
there was never anything strange about rodent weights, we were simply looking at several species at once.

::::::::::::::::::::::::::::::::::::: challenge

### Challenge 4: Finding unusual individuals

Using `surveys_centered`, find the ten animals that are furthest above their species' mean weight.
Which species do they belong to?

**Hint:** `arrange()` and `head()` will both be useful.
 
:::::::::::::::::::::::: solution


``` r
surveys_centered %>% 
  arrange(desc(weight_diff)) %>% 
  head(10)
```

``` output
# A tibble: 10 × 5
# Groups:   species_id [1]
   species_id sex   weight mean_weight weight_diff
   <chr>      <chr>  <dbl>       <dbl>       <dbl>
 1 NL         M        278        158.        120.
 2 NL         M        275        158.        117.
 3 NL         F        274        158.        116.
 4 NL         M        270        158.        112.
 5 NL         M        269        158.        111.
 6 NL         F        264        158.        106.
 7 NL         M        260        158.        102.
 8 NL         M        260        158.        102.
 9 NL         M        259        158.        101.
10 NL         M        259        158.        101.
```

These are the individuals most unusual relative to their own species, which is a different question from "which animals are heaviest".
A very heavy NL might be entirely typical for an NL, while a moderately heavy DM could be a remarkable outlier.

In a real analysis these are exactly the records you would want to inspect, since extreme values are sometimes genuine biological variation and sometimes data entry errors.

::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

## Working with dates

Our dataset spans 1977 to 1989.
A natural next question is whether these animals were getting bigger or smaller over that period.
To look at anything over time, we need a proper date.

You'll notice our data keeps year, month, and day in separate columns.
This is deliberate and generally good practice for stored data, because programs like Excel do automatic and sometimes irreversible things to anything that looks like a date.
But now that we are in R, we can safely assemble a date column.

First we paste the three pieces together.
The `paste()` function joins values, with `sep` controlling the character placed between them:


``` r
surveys_weight %>% 
  mutate(date = paste(year, month, day, sep = "-"))
```

``` output
# A tibble: 15,186 × 9
   species_id sex   plot_id  year month   day weight hindfoot_length date     
   <chr>      <chr>   <dbl> <dbl> <dbl> <dbl>  <dbl>           <dbl> <chr>    
 1 DM         M           3  1977     8    19     40              35 1977-8-19
 2 DM         M           7  1977     8    19     48              37 1977-8-19
 3 DM         F           4  1977     8    19     29              34 1977-8-19
 4 DM         F           4  1977     8    19     46              35 1977-8-19
 5 DM         M           7  1977     8    19     36              35 1977-8-19
 6 DO         F           8  1977     8    19     52              32 1977-8-19
 7 PF         M           2  1977     8    19      8              15 1977-8-19
 8 OX         F           3  1977     8    19     22              21 1977-8-19
 9 DM         F           7  1977     8    19     35              36 1977-8-19
10 PF         M           8  1977     8    19      7              12 1977-8-19
# ℹ 15,176 more rows
```

Our new column is created at the far end of the data.frame, so it doesn't print.

::::::::::::::::::::::::: spoiler

We can use relocate() to move columns inside our data.frame:


``` r
surveys_weight %>% 
  mutate(date = paste(year, month, day, sep = "-")) %>% 
  relocate(date, .after = day)
```

``` output
# A tibble: 15,186 × 9
   species_id sex   plot_id  year month   day date      weight hindfoot_length
   <chr>      <chr>   <dbl> <dbl> <dbl> <dbl> <chr>      <dbl>           <dbl>
 1 DM         M           3  1977     8    19 1977-8-19     40              35
 2 DM         M           7  1977     8    19 1977-8-19     48              37
 3 DM         F           4  1977     8    19 1977-8-19     29              34
 4 DM         F           4  1977     8    19 1977-8-19     46              35
 5 DM         M           7  1977     8    19 1977-8-19     36              35
 6 DO         F           8  1977     8    19 1977-8-19     52              32
 7 PF         M           2  1977     8    19 1977-8-19      8              15
 8 OX         F           3  1977     8    19 1977-8-19     22              21
 9 DM         F           7  1977     8    19 1977-8-19     35              36
10 PF         M           8  1977     8    19 1977-8-19      7              12
# ℹ 15,176 more rows
```

::::::::::::::::::::::::::::::::::::

Now we can see a column containing something that looks like a date, but it is just a character string.
R does not yet know it represents a point in time, so it cannot order it properly or space it correctly on an axis.
To make it a real date, we use the `ymd()` function from the `lubridate` package (already loaded with `tidyverse`), which parses a string in year-month-day order:


``` r
surveys_dates <- surveys_weight %>% 
  mutate(date = paste(year, month, day, sep = "-"),
         date = ymd(date))

surveys_dates
```

``` output
# A tibble: 15,186 × 9
   species_id sex   plot_id  year month   day weight hindfoot_length date      
   <chr>      <chr>   <dbl> <dbl> <dbl> <dbl>  <dbl>           <dbl> <date>    
 1 DM         M           3  1977     8    19     40              35 1977-08-19
 2 DM         M           7  1977     8    19     48              37 1977-08-19
 3 DM         F           4  1977     8    19     29              34 1977-08-19
 4 DM         F           4  1977     8    19     46              35 1977-08-19
 5 DM         M           7  1977     8    19     36              35 1977-08-19
 6 DO         F           8  1977     8    19     52              32 1977-08-19
 7 PF         M           2  1977     8    19      8              15 1977-08-19
 8 OX         F           3  1977     8    19     22              21 1977-08-19
 9 DM         F           7  1977     8    19     35              36 1977-08-19
10 PF         M           8  1977     8    19      7              12 1977-08-19
# ℹ 15,176 more rows
```


The date column now has type `<date>`.
Note that we used two separate lines inside one `mutate()`:
the first creates date as a character string, and the second overwrites it with the parsed version.

Now we can answer our question.
We'll look at the mean weight per year for a few of the most abundant species, so the result isn't dominated by rare species with noisy means:


``` r
surveys_dates %>% 
  filter(species_id %in% c("DM", "DS", "NL")) %>% 
  group_by(year, species_id) %>% 
  summarize(mean_weight = mean(weight)) %>% 
  ggplot(aes(x = year, y = mean_weight, color = species_id)) +
  geom_line()
```

``` output
`summarise()` has regrouped the output.
ℹ Summaries were computed grouped by year and species_id.
ℹ Output is grouped by year.
ℹ Use `summarise(.groups = "drop_last")` to silence this message.
ℹ Use `summarise(.by = c(year, species_id))` for per-operation grouping
  (`?dplyr::dplyr_by`) instead.
```

<img src="fig/working-with-data-rendered-unnamed-chunk-35-1.png" alt="Time-series plot of the weight of the DM, DS, and DL species between 1977 and 1989" width="600" height="600" style="display: block; margin: auto;" />

There is no dramatic trend, but there is clear year-to-year fluctuation, and the three species stay well separated throughout. That separation is the same finding as before, now visible across time: species identity is the dominant determinant of body size in this dataset.

::::::::::::::::::::::::::::::::::::: challenge

### Challenge 5: Making a time series
Because `ggplot()` takes data as its first argument, you can pipe data straight into it.
Build a pipeline from `surveys_dates` that counts the total number of animals caught on each day, and plots that count over time.

:::::::::::::::::::::::: solution


``` r
surveys_dates %>% 
  group_by(date) %>% 
  summarize(n = n()) %>%
  ggplot(aes(x = date, y = n)) +
  geom_line()
```

<img src="fig/working-with-data-rendered-unnamed-chunk-36-1.png" alt="" width="600" height="600" style="display: block; margin: auto;" />

There is a shorter way to write the counting step.
The `count()` function is a convenient wrapper for `group_by() + summarize(n = n())`:


``` r
surveys_dates %>% 
  count(date)
```

``` output
# A tibble: 312 × 2
   date           n
   <date>     <int>
 1 1977-08-19    15
 2 1977-08-20    24
 3 1977-08-21     2
 4 1977-09-11    20
 5 1977-09-12    17
 6 1977-09-13    11
 7 1977-10-16    30
 8 1977-10-17    26
 9 1977-10-18     9
10 1977-11-12    28
# ℹ 302 more rows
```

::::::::::::::::::::::::

Now split that daily count by sex, so you can see the number of each sex caught over time.
You will need to decide how to distinguish the sexes visually.

:::::::::::::::::::::::: solution


``` r
surveys_dates %>% 
  count(date, sex) %>% 
  ggplot(aes(x = date, y = n, color = sex)) +
  geom_line()
```

<img src="fig/working-with-data-rendered-unnamed-chunk-38-1.png" alt="" width="600" height="600" style="display: block; margin: auto;" />

Mapping sex to color gives us one line per sex. Notice that NA appears as its own line, since animals with unrecorded sex are still real captures.

Faceting by sex, or summarizing by month or year instead of day, would make the pattern easier to read.

::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

## Reshaping data

We have our answer.
The last task is to turn it into something we can hand to someone else.

Suppose a collaborator wants to know whether the size difference between species holds across the study area, or whether some plots have unusually large or small animals.
They don't use R, so they want a spreadsheet with species as rows and plots as columns.

We can start with the familiar split-apply-combine approach, grouping by both species and plot:


``` r
sp_by_plot <- surveys_weight %>% 
  group_by(species_id, plot_id) %>% 
  summarize(mean_weight = mean(weight)) %>%
  arrange(species_id, plot_id)
```

``` output
`summarise()` has regrouped the output.
ℹ Summaries were computed grouped by species_id and plot_id.
ℹ Output is grouped by species_id.
ℹ Use `summarise(.groups = "drop_last")` to silence this message.
ℹ Use `summarise(.by = c(species_id, plot_id))` for per-operation grouping
  (`?dplyr::dplyr_by`) instead.
```

``` r
sp_by_plot
```

``` output
# A tibble: 300 × 3
# Groups:   species_id [18]
   species_id plot_id mean_weight
   <chr>        <dbl>       <dbl>
 1 BA               3         8  
 2 BA              21         6.5
 3 DM               1        42.7
 4 DM               2        42.6
 5 DM               3        41.2
 6 DM               4        41.9
 7 DM               5        42.6
 8 DM               6        42.1
 9 DM               7        43.2
10 DM               8        43.4
# ℹ 290 more rows
```

That contains everything we need, but it is 300 rows long and comparing plots means scrolling up and down.
This format, one row per observation with a single value column, is called long format.
What our collaborator wants is wide format: one row per species, one column per plot.

The `tidyr` (already loaded with `tidyverse`) package has a pair of functions for moving between these shapes. 
`pivot_wider()` makes data wider, increasing the number of columns and reducing the number of rows. 
`pivot_longer()` does the opposite.

### Making data wider with `pivot_wider()`

`pivot_wider()` takes three main arguments:

- the data.frame
- `names_from`: which column should supply the names of the new columns?
- `values_from`: which column should supply the values that fill them?

Any column not used for `names_from` or `values_from` is left alone, and becomes the identifier for each row.

![](fig/pivot_wider.png){alt='Diagram depicting the behavior of `pivot_wider()` on a small tabular dataset.'}

In our case the new column names come from `plot_id` and the values come from `mean_weight`, which leaves `species_id` to identify the rows:


``` r
sp_by_plot_wide <- sp_by_plot %>% 
  pivot_wider(names_from = plot_id, 
              values_from = mean_weight)

sp_by_plot_wide
```

``` output
# A tibble: 18 × 25
# Groups:   species_id [18]
   species_id    `3`   `21`    `1`    `2`    `4`   `5`    `6`   `7`    `8`
   <chr>       <dbl>  <dbl>  <dbl>  <dbl>  <dbl> <dbl>  <dbl> <dbl>  <dbl>
 1 BA           8      6.5   NA     NA     NA     NA    NA     NA    NA   
 2 DM          41.2   41.5   42.7   42.6   41.9   42.6  42.1   43.2  43.4 
 3 DO          42.7   NA     50.1   50.3   46.8   50.4  49.0   52    49.2 
 4 DS         128.    NA    129.   125.   118.   111.  114.   126.  128.  
 5 NL         171.   136.   154.   171.   164.   192.  176.   170.  134.  
 6 OL          32.1   28.6   35.5   34     33.0   32.6  31.8   NA    30.3 
 7 OT          24.1   24.1   23.7   24.9   26.5   23.6  23.5   22    24.1 
 8 OX          22     NA     NA     22     NA     20    NA     NA    NA   
 9 PE          22.7   19.6   21.6   22.0   NA     21    21.6   22.8  19.4 
10 PF           7.12   7.23   6.57   6.89   6.75   7.5   7.54   7     6.78
11 PH          28     31     NA     NA     NA     29    NA     NA    NA   
12 PM          20.1   23.6   23.7   23.9   NA     23.7  22.3   23.4  23   
13 PP          17.1   13.6   14.3   16.4   14.8   19.8  16.8   NA    13.9 
14 RF          14.8   17     NA     16     NA     14    12.1   13    NA   
15 RM          10.3    9.89  10.9   10.6   10.4   10.8  10.6   10.7   9   
16 SF          NA     49     NA     NA     NA     NA    NA     NA    NA   
17 SH          76.0   79.9   NA     88     NA     82.7  NA     NA    NA   
18 SS          NA     NA     NA     NA     NA     NA    NA     NA    NA   
# ℹ 15 more variables: `9` <dbl>, `10` <dbl>, `11` <dbl>, `12` <dbl>,
#   `13` <dbl>, `14` <dbl>, `15` <dbl>, `16` <dbl>, `17` <dbl>, `18` <dbl>,
#   `19` <dbl>, `20` <dbl>, `22` <dbl>, `23` <dbl>, `24` <dbl>
```

Now a collaborator can read across a row to compare plots, or down a column to compare species.

### Why are there so many NAs?
A lot of NA values have appeared.
These are worth understanding rather than ignoring.

A data.frame must have a value in every row and column, so when a species-plot combination has no data, `pivot_wider()` has to put something there, and it inserts NA.
We can verify this.
Looking at the table, species BA has NA in plot 1, so let's go back to the long version and search for that combination:

sp_by_plot %>% 
  filter(species_id == "BA" & plot_id == 1)
Zero rows. There is no mean weight for BA in plot 1, either because that species was never caught there, or because every BA caught there had a missing weight and was removed by our `filter()` back in step 1.

So these NAs are not a problem with our pivot, they are a result.
They tell us that most species are not found in most plots, which is itself an ecologically meaningful pattern
Long format hides this, because absent combinations are simply missing rows.
Wide format makes it visible.

### The reverse operation: pivot_longer()
Data often arrives from spreadsheets in wide format, with a column per site, per treatment, or per time point
Most `tidyverse` tools, and `ggplot2` in particular, want long format, where each variable has its own column.
So you will frequently need to go the other way.

`pivot_longer()` takes `cols` for the columns you want to pivot, `names_to` for the name of the new column that will hold the old column names, and `values_to` for the name of the new column that will hold the values.

![](fig/pivot_longer.png){alt='Diagram depicting the behavior of `pivot_longer()` on a small tabular dataset.'}

Suppose our collaborator sent the wide table back to us and we wanted to plot it.
We would pivot every column except `species_id` back into long format:


``` r
sp_by_plot_wide %>% 
  pivot_longer(cols = -species_id, 
               names_to = "plot_id", 
               values_to = "mean_weight") %>% 
  filter(!is.na(mean_weight))
```

``` output
# A tibble: 300 × 3
# Groups:   species_id [18]
   species_id plot_id mean_weight
   <chr>      <chr>         <dbl>
 1 BA         3               8  
 2 BA         21              6.5
 3 DM         3              41.2
 4 DM         21             41.5
 5 DM         1              42.7
 6 DM         2              42.6
 7 DM         4              41.9
 8 DM         5              42.6
 9 DM         6              42.1
10 DM         7              43.2
# ℹ 290 more rows
```

We filter out the NAs because they were created by the pivot and don't represent real observations. With that done, we are back to the data we started with, and it is ready to plot:


``` r
sp_by_plot_wide %>% 
  pivot_longer(cols = -species_id, 
               names_to = "plot_id", 
               values_to = "mean_weight") %>% 
  filter(!is.na(mean_weight)) %>% 
  ggplot(aes(x = plot_id, y = species_id, fill = mean_weight)) +
  geom_tile()
```

<img src="fig/working-with-data-rendered-unnamed-chunk-42-1.png" alt="" width="600" height="600" style="display: block; margin: auto;" />

This is a nice summary of the whole analysis.
Each row is a species and the color is its mean weight, so the horizontal bands of consistent color show that species identity determines body size, while the gaps show which species occur where.

## Exporting our results

Our collaborator is waiting, so let's write the wide table to a CSV file.

First, those bare numeric column names are not very informative in a spreadsheet, and some programs behave oddly with column names that are just numbers.
Conveniently, `pivot_wider()` has a names_prefix argument:


``` r
surveys_sp <- sp_by_plot %>% 
  pivot_wider(names_from = plot_id, 
              values_from = mean_weight,
              names_prefix = "plot_")

surveys_sp
```

``` output
# A tibble: 18 × 25
# Groups:   species_id [18]
   species_id plot_3 plot_21 plot_1 plot_2 plot_4 plot_5 plot_6 plot_7 plot_8
   <chr>       <dbl>   <dbl>  <dbl>  <dbl>  <dbl>  <dbl>  <dbl>  <dbl>  <dbl>
 1 BA           8       6.5   NA     NA     NA      NA    NA      NA    NA   
 2 DM          41.2    41.5   42.7   42.6   41.9    42.6  42.1    43.2  43.4 
 3 DO          42.7    NA     50.1   50.3   46.8    50.4  49.0    52    49.2 
 4 DS         128.     NA    129.   125.   118.    111.  114.    126.  128.  
 5 NL         171.    136.   154.   171.   164.    192.  176.    170.  134.  
 6 OL          32.1    28.6   35.5   34     33.0    32.6  31.8    NA    30.3 
 7 OT          24.1    24.1   23.7   24.9   26.5    23.6  23.5    22    24.1 
 8 OX          22      NA     NA     22     NA      20    NA      NA    NA   
 9 PE          22.7    19.6   21.6   22.0   NA      21    21.6    22.8  19.4 
10 PF           7.12    7.23   6.57   6.89   6.75    7.5   7.54    7     6.78
11 PH          28      31     NA     NA     NA      29    NA      NA    NA   
12 PM          20.1    23.6   23.7   23.9   NA      23.7  22.3    23.4  23   
13 PP          17.1    13.6   14.3   16.4   14.8    19.8  16.8    NA    13.9 
14 RF          14.8    17     NA     16     NA      14    12.1    13    NA   
15 RM          10.3     9.89  10.9   10.6   10.4    10.8  10.6    10.7   9   
16 SF          NA      49     NA     NA     NA      NA    NA      NA    NA   
17 SH          76.0    79.9   NA     88     NA      82.7  NA      NA    NA   
18 SS          NA      NA     NA     NA     NA      NA    NA      NA    NA   
# ℹ 15 more variables: plot_9 <dbl>, plot_10 <dbl>, plot_11 <dbl>,
#   plot_12 <dbl>, plot_13 <dbl>, plot_14 <dbl>, plot_15 <dbl>, plot_16 <dbl>,
#   plot_17 <dbl>, plot_18 <dbl>, plot_19 <dbl>, plot_20 <dbl>, plot_22 <dbl>,
#   plot_23 <dbl>, plot_24 <dbl>
```

Now we can save it with `write_csv()` from the `readr` package (already loaded with `tidyverse`). The first argument is the data.frame, the second is the path to the file we want to create, including the .csv extension:


``` r
write_csv(surveys_sp, "data/cleaned/surveys_meanweight_species_plot.csv")
```

If we look in our data/cleaned folder, the new file is there, ready to send.

:::::::::::::::::::::::::::::::::::::::::: callout

### Where should output go?

We wrote this file into data/cleaned alongside our input data, which is fine for a small project.
In a larger one, you might prefer a separate results/ or output/ folder, so it stays obvious which files you received and which files you produced.

Whatever you choose, the important thing is that you never overwrite your raw data.
Our original `surveys_complete_77_89.csv` is exactly as it was when we started, and every change we made to it exists as code in our script.

::::::::::::::::::::::::::::::::::::::::::

## Putting it all together

We have covered a lot of functions, and it can be hard to see how they fit together when you meet them one at a time.
So let's step back and look at the whole analysis as a single piece of code.

Here is everything we did to answer our research question, from reading the file to writing the result:


``` r
library(tidyverse)

# Read the raw data
surveys <- read_csv("data/cleaned/surveys_complete_77_89.csv")
```

``` output
Rows: 16878 Columns: 13
── Column specification ────────────────────────────────────────────────────────
Delimiter: ","
chr (6): species_id, sex, genus, species, taxa, plot_type
dbl (7): record_id, month, day, year, plot_id, hindfoot_length, weight

ℹ Use `spec()` to retrieve the full column specification for this data.
ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

``` r
# 1. Narrow to the rows and columns we need
surveys_weight <- surveys %>% 
  filter(!is.na(weight)) %>% 
  select(species_id, sex, plot_id, year, month, day, weight, hindfoot_length)

# 2. Test the hypothesis: is weight structured by species?
species_weights <- surveys_weight %>% 
  group_by(species_id) %>% 
  summarize(mean_weight = mean(weight),
            n = n()) %>% 
  arrange(desc(mean_weight)) %>% 
  ungroup()

# 3. Remove the species effect to see what variation is left
surveys_centered <- surveys_weight %>% 
  group_by(species_id) %>% 
  mutate(mean_weight = mean(weight),
         weight_diff = weight - mean_weight) %>% 
  ungroup()

# 4. Add a date column and look at change over time
surveys_dates <- surveys_weight %>% 
  mutate(date = ymd(paste(year, month, day, sep = "-"))) %>% 
  relocate(date, .after = day)

# 5. Build a shareable summary table and export it
surveys_sp <- surveys_weight %>% 
  summarize(mean_weight = mean(weight),
            .by = c(species_id, plot_id)) %>% # This replaces group_by(species_id, plot_id)
  ungroup() %>% 
  pivot_wider(names_from = plot_id, 
              values_from = mean_weight,
              names_prefix = "plot_")

write_csv(surveys_sp, "data/cleaned/surveys_meanweight_species_plot.csv")
```

That's the whole analysis.
Around twenty lines of code, readable from top to bottom, and it would run identically on a colleague's computer or on your own in two years' time.

Notice the shape of it, because this shape recurs in almost every analysis you will ever do:

1. Read your data, and leave the original file untouched.
2. Subset down to the rows and columns your question needs.
3. Create any variables your question requires but your data doesn't have. This could also involve cleaning the data.
4. Group and summarize to compare parts of your data against each other.
5. Reshape the result into whatever form your audience or your next tool expects.
6. Export it.

The functions will change depending on what you're doing, and there are hundreds more in the tidyverse that we haven't touched. 
But this sequence stays the same, and recognizing which step you are on is often the fastest way to work out which function you need.

### What did we find out?
We began with a puzzle: the distribution of rodent weights had two humps, and the heaviest 10% of animals weighed thirty times more than the lightest 10%.

Using split-apply-combine, we found that the dataset contains species with mean weights ranging from under 10 grams to over 150 grams, and that the abundant species cluster at the two ends of that range rather than spreading evenly across it.
Two species in particular, DM with over 5000 captures averaging around 42 grams, and DS and NL with thousands of captures averaging well over 100 grams, dominate the two humps.

When we subtracted each species' own mean weight from every individual, the bimodality disappeared entirely, leaving a single distribution centered on zero.
That's about as clear a confirmation as you could ask for:
there is nothing unusual about rodent body weights in this dataset.
We were looking at several different animals at once and treating them as one.

This is worth sitting with for a moment, because it is a genuinely common experience in data analysis.
The strange pattern was not a biological phenomenon, a measurement error, or an outlier problem.
It was an artefact of aggregating over a variable that mattered.
Whole-dataset summaries like `summary()` and `quantile()` are excellent for spotting that something is going on, but they cannot tell you what, because by design they collapse exactly the structure you need to see.
The moment you can split your data into meaningful groups, questions that looked mysterious often become straightforward.

::::::::::::::::::::::::::::::::::::: challenge

### Challenge 6: Your own analysis

You now have the tools to carry out a small analysis from start to finish.
Pick one of the questions below, or create your own, and build a pipeline to answer it.
Then write one or two sentences stating what you found.

- Last episode we plotted mean `hindfoot_length` by `sex` and saw a difference. Is that difference still there once you account for species? (Hint: this is the same problem we just solved for weight.)
- Which plots have the greatest number of distinct species? Does that change over the years? (`n_distinct()` will be useful.)
- Is there a relationship between `weight` and `hindfoot_length`? Does the relationship differ between species?
- Are animals of unrecorded sex systematically different from the rest? Which species are they concentrated in, and does that explain anything?

:::::::::::::::::::::::: solution

There is no single right answer here, but as an illustration of the first question:


``` r
surveys %>% 
  filter(!is.na(hindfoot_length) & !is.na(sex)) %>% 
  filter(species_id %in% c("DM", "DO", "PP", "RM")) %>% 
  group_by(species_id, sex) %>% 
  summarize(mean_hindfoot = mean(hindfoot_length),
            n = n()) %>% 
  ungroup() %>% 
  ggplot(aes(x = sex, y = mean_hindfoot, fill = sex)) +
  geom_col() +
  facet_wrap(vars(species_id))
```

``` output
`summarise()` has regrouped the output.
ℹ Summaries were computed grouped by species_id and sex.
ℹ Output is grouped by species_id.
ℹ Use `summarise(.groups = "drop_last")` to silence this message.
ℹ Use `summarise(.by = c(species_id, sex))` for per-operation grouping
  (`?dplyr::dplyr_by`) instead.
```

<img src="fig/working-with-data-rendered-unnamed-chunk-46-1.png" alt="" width="600" height="600" style="display: block; margin: auto;" />

The differences between the sexes within each species are small, certainly much smaller than the differences between species.
Just as with weight, the apparent effect of sex in the whole-dataset plot was largely driven by differences in which species each sex was sampled from, rather than by a large biological difference between males and females.

::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::

## Where to go next

A few things that follow naturally from here, if you want to keep learning:

- Joining data. We worked with one table, but real projects usually have several. dplyr's `left_join()` and friends combine tables by matching on shared columns, which is how you would attach the species names from a separate species table to our species_id column.
- More `dplyr` verbs. `slice_max()`, `across()`, `case_when()`, and `rowwise()` all solve common problems more neatly than what we've shown.
- Data cleaning. The file we used was already cleaned for you. The `janitor` package and `stringr` (for text) are invaluable when yours isn't.
- R for Data Science (available free at <r4ds.hadley.nz>) covers everything in this workshop in much greater depth, and is the standard next step.

::::::::::::::::::::::::::::::::::::: keypoints 

- use `filter()` to subset rows and `select()` to subset columns
- build up pipelines one step at a time before assigning the result
- it is often best to keep components of dates separate until needed, then use `mutate()` to make a date column
- `group_by()` can be used with `summarize()` to collapse rows or `mutate()` to keep the same number of rows
- `pivot_wider()` and `pivot_longer()` are powerful for reshaping data, but you should plan out how to use them thoughtfully

::::::::::::::::::::::::::::::::::::::::::::::::
