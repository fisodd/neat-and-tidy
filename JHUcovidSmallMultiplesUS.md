---
title: "Clean and Tidy: Using Tidyverse to Manage COVID-19 Data"
subtitle: "Generating Small-Multiple Charts of Cases in the US Using the JHU CSSE COVID-19 Dataset"
author: "Alexander Carlton"
date: "2020-04-09"
output:
    tufte::tufte_html:
        tufte_variant: "envisioned"
---



# Introduction

COVID-19 has surely wreaked havok all over,
but it has also demonstrated the value of data --
especially well managed, timely data.

The good folks at the Johns Hopkins University 
Center for Systems Science and Engineering (JHU CSSE)
have provided one of the most obvious early examples
of an immediately useful dataset -- and wonderfully
they have
[shared it all openly](https://github.com/CSSEGISandData/COVID-19)
so that everyone can work with the numbers
until we each find the best ways to make sense of all this.


```marginfigure
[Neat and Tidy](https://www.fisodd.com/code/neat-and-tidy)
is a place where I have collected a variety of lessons I have
learned as have explored the power of the
[Tidyverse](https://www.tidyverse.org/).
```

This write-up comes from one of my own learning exercises
using the powerful tools of the
`tidyverse` library for the `R` environment.
The author is lifelong student who has recently found the Tidyverse
to be an effective means of working with and displaying this data.
There are no insightful analyses here, this is just a review of 
one attempt to use tidyverse functions to do good work
with an interesting dataset.

The original goal was to compare how my local community
(one of the earliest with confirmed cases of COVID-19) 
was comparing to others around the country.
The intention was to build a 
[small-multiple](https://en.wikipedia.org/wiki/Small_multiple)
graphic that
allowed me to compare caseload curves and also appreciate the 
differences (if any) between large and small communities.


# Setup

There are some assumptions built into any working code.
Perhaps the details of the setup used here
can provide some idea of the assumptions
that underlie this code.

## Configuration

The values below can be configured to adapt to the local needs.

### Local Files

The first set of variables are holding where to find a local copy of
the dataset.


```marginfigure
The method used here was to clone the 
[COVID-19 repository](https://github.com/CSSEGISandData/COVID-19)
provided by the JHU CSSE team.
Cloning the repository results in a directory hierarchy of files
underneath the `COVID-19` directory.
```


```r
# The following are based on the local copy of the data
jhu_directory <- "COVID-19/csse_covid_19_data/csse_covid_19_time_series/"
jhu_cases_filename <- "time_series_covid19_confirmed_US.csv"
jhu_death_filename <- "time_series_covid19_deaths_US.csv"
```

### Display Parameters

In this case we are defining a couple of variables
that can be used to adjust the details of the resulting chart.


```r
min_pop <- 10000      # can be useful to exclude tiny entities with wide variances
max_counties <- 25    # how many facets at most, square values look best
```

## Environment

The `tidyverse` library is the main dependency.
`lubridate` is used so we can `lag()` to fetch values from the day before.
The `scales` library is used for a minor touch 
to make the legend a tiny bit easier to read.


```marginfigure
"If I have seen further it is by standing on the shoulders of Giants"
-- Issac Newton
```


```r
library(tidyverse)
library(lubridate)
library(scales)
```


# Inputs

This particular analysis is based on the time-series
provided for the US, which has day by day values for
cumulative confirmed cases categorized by state and county.

These files have several columns which define or label the counties,
and then a bunch of columns for the values (one column for each date).


```r
jhu_cols <- cols(
    .default = col_double(),
    iso2 = col_character(),
    iso3 = col_character(),
    Admin2 = col_character(),
    Province_State = col_character(),
    Country_Region = col_character(),
    Combined_Key = col_character()
)
```

To make this data useful for the tidyverse functions,
once it is read in we transform into a "tidy" data format.


```marginfigure
Some may find the [tidy](https://r4ds.had.co.nz/) 
approach awkward, but in an environment with many (sometimes
arcane) methods to achieve any result, the `tidyverse` stands 
out as a sane system whose methods can be surprising effective.
However, I will admit the new 
[pivot](https://tidyr.tidyverse.org/dev/articles/pivot.html)
functions make working in/out of tidy data a whole lot easier
than the earlier "gather" and "spread" functions -- and
for me this makes the entire `tidyverse` much easier to approach.
```


```r
cases_data <- read_csv(
    paste0(jhu_directory, jhu_cases_filename),
    col_types = jhu_cols
) %>%
    select(
        -UID, -iso2, -iso3, -code3,
        -Lat, -Long_,
        Country = "Country_Region",
        State = "Province_State",
        County = Admin2
    ) %>%
    select(-Country) %>%  # Country always == "US" in this dataset
    pivot_longer(
        -c(FIPS, County, State, Combined_Key),
        names_to = "date_str",
        values_to = "Cases"
    ) %>%
    mutate(
        Date = as.Date(date_str, "%m/%d/%y"),
        date_str = NULL
    )
```

We perform the same steps on a related file
which contains the cumulative counts of COVID-19 related deaths.

*Note:* the deaths file contains a useful extra column
representing the population for each county in the dataset;
and we certainly want to keep and use that information.


```r
death_data <- read_csv(
    paste0(jhu_directory, jhu_death_filename),
    col_types = jhu_cols
) %>%
    select(
        -UID, -iso2, -iso3, -code3,
        -Lat, -Long_,
        Country = "Country_Region",
        State = "Province_State",
        County = Admin2
    ) %>%
    select(-Country) %>%  # Country always == "US" in this dataset
    pivot_longer(
        -c(FIPS, County, State, Combined_Key, Population),
        names_to = "date_str",
        values_to = "Death"
    ) %>%
    mutate(
        Date = as.Date(date_str, "%m/%d/%y"),
        date_str = NULL
    )
```


# Operate

There is not much analysis in this exercise, so the operations
we have are really just to use the many tools in the `tidyverse`
to strip all this data down to the chart we seek.

# Join Datasets

Joining these datasets together
enables us to use the same population data for both cases and deaths,
and also leads us to a simple way to include both on the same chart.


```r
dataset <- cases_data %>%
    left_join(
        death_data, 
        by = c("FIPS", "County", "State", "Combined_Key", "Date")
    ) %>%
    rename(
        Label = "Combined_Key"
    )
```


# New Cases

First, we use the `lubridate` tools to calculate values for
"new cases" -- the counts of cases new to each day,
or in other words the difference in cumulative count
between today and yesterday.

We `arrange` the data so that the `lag()` calls will fetch
the preceeding day's entry rather than yank a value from
whatever item happened to fall into the preceeding row.


```r
workset <- dataset %>%
    arrange(
        State, County, Date
    ) %>%
    mutate(
        NewCases = if_else(
            County == lag(County, 1, default = ""), 
            Cases - lag(Cases, 1, default = 0),  
            NA_real_
        ),
        NewDeath = if_else(
            County == lag(County, 1, default = ""),
            Death - lag(Death, 1, default = 0),
            NA_real_
        )
    ) 
```

The `if_else()` logic is there to prevent us from `lag()`
returning a value that comes from a different location.
And to avoid a possible `NA` on the first row (where there
is no data for `lag()` to fetch), we provide a default lag value
[but we really don't care what that value is so long as it
doesn't match a recent date].

## Where Does One Find a County Called "Unassigned"?

One of the gotchas with this dataset is that not all the
cases are assigned to a county (or any other entity where
we have a known population).


```marginfigure
Sigh.  Welcome to the real world.  
The good folks at JHU CSSE are delivering an amazing resource here, 
a few workarounds is a very small price to pay for all this.
```

Due to lapses in bookkeeping or whatever reason many states
have reports of cases where the county is listed only as
"Unassigned".
Also, in other situations, states are loathe to consider
visitors who are being treated in their state as part of
their state's afflicted population.  Hence there are a number
of cases listed as "Out of XX".

This is a problem for our population-based analysis
since the known population is 0, and dividing by zero
makes a mess of our results.


```marginfigure
This does affect the accuracy of our analyses, but since our
goal here is only a rough comparison chart what accuracy we still
have should be sufficient to find if things are near or far
from each other.  We would want to be more careful if our goal
was dependent on the need to extract exact values.

Like most real-world datasets, the accuracy of the source data
does not support the precision built into our calculation tools.
With the available COVID-19 testing results, there is already
evidence anyway that the reported values of confirmed cases are
only some unknown percentage of the  real numbers of people
who have caught the disease (and may not know it).
This exercise here may be sufficient for rough comparisons
between these counties, but one needs to consider differences
in conditions, environment, and reporting rules before 
one attempts to draw specific conclusions from these materials.
```

The workaround used here is to sum up these "extra" cases
state by state and later add them to whichever county has
the highest count of cases.
The hope is that this county is possibly the
most likely location where these cases were found; the
likely outcome is that these extra cases become a small
bit of noise in the data -- the worst case is that this
workaround will cause some county-level data on at least
some dates to be notably distorted.


```r
outs_data <- workset %>%
    filter(
        County == "Unassigned" | str_detect(County, "Out of ")
    ) %>%
    group_by(State, Date) %>%
    summarise(
        ExCases = sum(NewCases),
        ExDeath = sum(NewDeath)
    )
```


## Normalize by Population

We can now add these extras to our existing values
and then generate new values normalized per million
in population for that county -- so all these values
are now in terms of cases per million which makes for
easier cross comparisons between small and large
counties.


```r
workset <- workset %>%
    # join "outs" counts as new columns of state-level data
    left_join(outs_data, by = c("State", "Date")) %>%
    # Now we can drop out counties with 0 or very low Population
    # ... this will also drop out the "Unassigned" counties
    filter(
        Population > min_pop
    ) %>%
    arrange(State, Date, desc(Cases)) %>%
    # Once sorted, the first row for each date has the most Cases
    mutate(
        SumCases = if_else(
            Date != lag(Date, 1, default = 0),
            NewCases + ExCases,
            NewCases
        ),
        SumDeath = if_else(
            Date != lag(Date, 1, default = 0),
            NewDeath + ExDeath,
            NewDeath
        )
    ) %>%
    # Finally, calculate rates per million in population
    mutate(
        NormedCases = if_else(
            NewCases > 0,
            NewCases / (Population / 1000000),
            NA_real_
        ),
        NormedDeath = if_else(
            NewDeath > 0,
            NewDeath / (Population / 1000000),
            NA_real_
        )
    ) %>%
    select(
        -Cases, -Death,
        -NewCases, -NewDeath,
        -SumCases, -SumDeath,
        -ExCases, -ExDeath
    )
```


## Find the Counties with Highest Peaks

Rather than display the thousands of counties in the US,
this chart will be based on just a few of those with highest
rates (cases per million).
[The number of counties included is based on the
`max_counties` value set in the Configuration section
at the top of this file.]


```r
filter_keys <- workset %>%
    group_by(Label, Date) %>%
    summarise(peak = max(NormedCases)) %>%
    arrange(desc(peak)) %>%
    pull(Label) %>%
    unique() %>%
    head(max_counties)
```

## Final Details

Finally, we produce a minimal "viewset" with just the data we'll display --
`ggplot()` is already doing a lot of work, it may help to avoid
slogging a data frame replete with a wide variety of unneeded values.


```r
viewset <- workset %>%
    # Just include the countries in the desired list
    filter(
        Label %in% filter_keys
    ) %>%
    # Drop out any rows that won't display anyway
    filter(
        !is.na(NormedCases),
        NormedCases > 0.01
    ) %>%
    # Drop out any columns not displayed
    select(
        -FIPS, -County, -State
    )
```


# Display

Using the many features of ``ggplot()``
we can build a very complex chart
showing small-multiples of each counties' case rates over time.

The "Cases" data are the primary aesthetic and
will show up as sequences of colored dots on each chart, 
with the color representing the scale of the
population in that county.

The "Death" data is referenced in the second `geom_point()` 
and will show up as much smaller red dots
trailing underneath the "Cases" points.

The legend has been tweaked pretty seriously to enable a
tertiary comparison of the relative sizes of the counties
displayed.  This legend is based on a `log10` transform
of the `Population` column using the vivid Viridis
color scale -- reworking the scale by orders of magnitude
makes it is possible to get a reasonable sense of each county's
size even in the tiny subplots within the small-multiple view.
The legend itself has been hacked with calls to `guides()` and `theme()`
to render a long and thin legend along the bottom of the chart
which may make it easier to see how to read the
colors of the counties while taking only a bit of visual room.


```r
g <- ggplot(
    viewset,
    aes(
        x = Date,
        y = NormedCases,
        color = Population
    )
) +
    geom_point(na.rm = TRUE) +
    geom_point(
        aes(y = NormedDeath),
        size = 0.25,
        color = "red",
        na.rm = TRUE
    ) +
    scale_x_date(
        minor_breaks = NULL
    ) +
    scale_y_log10(
        labels = label_comma(accuracy = 1),
        minor_breaks = NULL
    ) +
    facet_wrap(~fct_relevel(Label, filter_keys)) +
    scale_color_viridis_c(
        option = "D",
        trans = "log10",
        labels = label_comma()
    ) +
    guides(
        color = guide_colorbar(
            barwidth = unit(0.75, "npc"),
            barheight = unit(0.01, "npc"),
            title = "County Population"
        )
    ) +
    labs(
        title = "COVID-19: Daily Counts of Confirmed Cases (and deaths) Normalized by Population",
        subtitle = paste(
            max_counties,
            "of the US Counties with the Highest Levels of Cases per Million in Population"
        ),
        x = NULL,
        y = "Daily Count of Cases (and deaths) per Million Population (log scale)",
        caption = paste(
            "COVID-19 Data from Johns Hopkins University CSSE as of",
            format(max(viewset$Date), "%A, %B %e, %Y")
        )
    ) +
    theme_minimal() +
    theme(
        axis.text.x = element_text(angle = 90, vjust = 0.5),
        legend.position = "bottom"
    )
print(g)
```

![plot of chunk chart](figure/chart-1.png)

Clearly this is a very busy graphic, but it helped address the question
I was investigating at the time (comparing how my local county compared
to other counties, large and small, regarding where we were on the curve).
Reviewing this chart over time can show which counties are still facing
the difficulties of rapid case growth and which appear to be experiencing
improving conditions.

And, perhaps more importantly, working through this exercise proved to be
a very good opportunity to become more familiar with several
tidyverse features that hadn't yet become a part of my usual processes.

I hope at least something here was helpful.
This and some of my other exercises are part of a
[Neat And Tidy](https://www.fisodd/code/neat-and-tidy) project
with code shared in [my GitHub](https://github.com/fisodd/neat-and-tidy).