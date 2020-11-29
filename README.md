## Source data

Source data is obtained at
<https://data.cdc.gov/NCHS/Provisional-COVID-19-Death-Counts-by-Sex-Age-and-W/vsak-wrfu>.

## Validation: Recreation of Briand’s Chart

To validate the data, first we recreate some of the exhibits in Briand’s
talk.

    library(httr)
    library(rlist)
    library(jsonlite)
    library(tidyverse)

    ## ── Attaching packages ─────────────────────────────────────── tidyverse 1.3.0 ──

    ## ✓ ggplot2 3.3.2     ✓ purrr   0.3.4
    ## ✓ tibble  3.0.4     ✓ dplyr   1.0.2
    ## ✓ tidyr   1.1.2     ✓ stringr 1.4.0
    ## ✓ readr   1.4.0     ✓ forcats 0.5.0

    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## x dplyr::filter()  masks stats::filter()
    ## x purrr::flatten() masks jsonlite::flatten()
    ## x dplyr::lag()     masks stats::lag()

    library(lubridate)

    ## 
    ## Attaching package: 'lubridate'

    ## The following objects are masked from 'package:base':
    ## 
    ##     date, intersect, setdiff, union

    library(viridis)

    ## Loading required package: viridisLite

    # Fetch data from CDC in JSON format
    url <- "https://data.cdc.gov/api/views/vsak-wrfu/rows.json"
    params <- list(accessType="DOWNLOAD")

    get <- GET(url = url, config=params)
    print(paste0("That worked OK? ", !http_error(get)))

    ## [1] "That worked OK? TRUE"

    data <- fromJSON(content(get, as="text"))

    table <- as_tibble(data$data)

    ## Warning: The `x` argument of `as_tibble.matrix()` must have unique column names if `.name_repair` is omitted as of tibble 2.0.0.
    ## Using compatibility `.name_repair`.
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_warnings()` to see where this warning was generated.

    names(table) <- data[["meta"]][["view"]][["columns"]][["name"]]

    final_table <- table %>%
      mutate(`COVID-19 Deaths` = as.numeric(`COVID-19 Deaths`),
             `Total Deaths` = as.numeric(`Total Deaths`)) %>% 
      select(12:16)

    plot.data <- final_table %>% 
      filter(`End Week` <= ymd(20200905)) %>%
      filter(`Sex` == "All Sex") %>%
      group_by(`End Week`, `Sex`,
        Age_Group_Regroup =
        if_else(`Age Group` %in% c("Under 1 year","1-4 years","5-14 years"),
                "14 and under", `Age Group`)) %>% 
      summarize(`Total Deaths` = sum(as.numeric(`Total Deaths`), na.rm=T),
                `COVID-19 Deaths` = sum(as.numeric(`COVID-19 Deaths`), na.rm=T)) %>% 
      ungroup()

    ## `summarise()` regrouping output by 'End Week', 'Sex' (override with `.groups` argument)

    plot.data.check <- plot.data %>% 
      group_by(`Sex`, `End Week`) %>% 
      mutate(td = sum(`Total Deaths`)) %>% 
      ungroup() %>% 
      filter(Age_Group_Regroup == "All Ages") %>% 
      mutate(check = td/2 - `Total Deaths`)

    plot <- plot.data %>% 
      filter(Age_Group_Regroup != "All Ages") %>% 
      mutate(`End Week` = as.Date(`End Week`)) %>% 
      group_by(`End Week`, `Sex`) %>% 
      mutate(rate = `Total Deaths`/sum(`Total Deaths`, na.rm=T)) %>% 
      ungroup() %>% 
      select(Age_Group_Regroup, rate, `End Week`) %>% 
      ggplot(mapping=aes(x=`End Week`, y=rate, 
                         fill=fct_rev(Age_Group_Regroup))) +
      geom_bar(position="stack", stat="identity", color="black", size=0.3) +
      theme_minimal() +
      scale_fill_viridis_d() +
      scale_y_continuous(label=scales::percent) +
      scale_x_date(labels = scales::date_format(format="%b %d"),
                   breaks = seq.Date(from=as.Date(min(plot.data$`End Week`)),
                                     to=as.Date(max(plot.data$`End Week`)),
                                     by="1 week")) +
      theme(axis.text.x = element_text(angle=90)) +
      labs(x="Week",
           y="Observed Proportion of All Deaths",
           title="Observed Proportion of Total Deaths by Week",
           caption="Source: CDC",
           fill="Age Cohort") 
           
    plot

![](README_files/figure-markdown_strict/validate-1.png) This chart lines
up very nicely with Briand’s version of the same presentation, so I’m
happy to tie off the validation portion of this exercise.

Let’s also take care to note that the preceding graph does *not*
describe the mortality rate broken out by cohort. Rather, it describes
the proportion of observed deaths out of the total number of observed
deaths, broken out by age cohort. These two things are very different
measurements, though a quick reading of the article in question appears
to (a) make a case for the lack of excess deaths observed in 2020 which
is, to be technical, totally nutso; and (b) sets up the argument that
there is some kind of misunderstanding of COVID mortality figures
because the total proportion of mortality by age group has not changed
over time. Again, trying to be totally charitable here, but I refer you
to the following direct article quotation:

> “These data analyses suggest that in contrast to most people’s
> assumptions, the number of deaths by COVID-19 is not alarming. In
> fact, it has relatively no effect on deaths in the United States.”

Let’s explore this assertion further.

Now we’ll look at another presentation of the data, where we take the
total overall mortality rate, and slice and dice that by age cohort, and
by time period, and look at this in data visualization again in
different ways - and you are free to modify and run this code to explore
different angles yourself!

Using the [https://wonder.cdc.gov/ucd-icd10.html](CDC%20Wonder) system
we can query mortality by the same age banded cohorts all the way back
to 1999. Use of this system requires you agree to not use the data for
bad purposes, so I can’t have the code pull it in automatically.
Nonetheless, the data used here is retrieved from the link above on Sat
Nov 28 2020 at around 6:30 PM. Finally, since death is seasonal over
time I’m only going to look at data on an annual basis, but split out
into the same age-banded cohorts.

    data2 <- read_delim("Underlying Cause of Death, 1999-2018.txt", "\t",
                        escape_double = FALSE, trim_ws = TRUE) %>% 
      select(Cohort=`Ten-Year Age Groups`, Year, Deaths, Population) %>% 
      drop_na() %>% 
      mutate(q_x = Deaths/Population)

    ## 
    ## ── Column specification ────────────────────────────────────────────────────────
    ## cols(
    ##   Notes = col_character(),
    ##   `Ten-Year Age Groups` = col_character(),
    ##   `Ten-Year Age Groups Code` = col_character(),
    ##   Year = col_double(),
    ##   `Year Code` = col_double(),
    ##   Deaths = col_double(),
    ##   Population = col_double(),
    ##   `Crude Rate` = col_double()
    ## )

    ## Warning: 46 parsing failures.
    ## row col  expected    actual                                       file
    ## 233  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018.txt'
    ## 234  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018.txt'
    ## 235  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018.txt'
    ## 236  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018.txt'
    ## 237  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018.txt'
    ## ... ... ......... ......... ..........................................
    ## See problems(...) for more details.

    plot2.data <- data2 %>% 
      group_by(Year, 
               Age_Group_Regroup = if_else(Cohort %in% c("< 1 year","1-4 years","5-14 years"), "14 and under", Cohort)) %>%
      summarize(Deaths = sum(Deaths)) %>% 
      mutate(prop = Deaths/sum(Deaths)) %>% 
      ungroup() 

    ## `summarise()` regrouping output by 'Year' (override with `.groups` argument)

    plot2 <- plot2.data %>% 
      ggplot(mapping=aes(x=as_factor(Year), y=prop,
                         fill=fct_rev(Age_Group_Regroup))) +
      geom_bar(position="stack", stat="identity", color="black", size=0.3) +
      theme_minimal() +
      scale_fill_viridis_d() +
      scale_y_continuous(label=scales::percent) +
      scale_x_discrete(breaks=seq(1999, 2018, 1)) +
      theme(axis.text.x = element_text(angle=90)) +
      labs(x="Year",
           y="Observed Proportion of All Deaths",
           title="Observed Proportion of Total Deaths by Year",
           caption="Source: CDC",
           fill="Age Cohort") 
           
    plot2

![](README_files/figure-markdown_strict/mortality-over-time-1.png) This
graph should look unsurprisingly similar to the one in the previous
section. The one difference we can see is that the larger time horizon
allows us to take into perspective the slowly shrinking 75-84 age band.
This absolutely does *not* mean fewer people are dying *in aggregate*.
In fact, due to population increases and other reasons, *more* people
are dying (judged on the basis of just raw death counts).

So, all this chart can tell us is that it appears more and more folks
are moving to the 85+ band or to other bands out of the 75-84 age band.
A concurrence of phenomena are at play here: people are are living
longer is just one bit of the answer.

The point of this discussion is that we can glean absolutely zero
information about total excess mortality in the US *in aggregate* from
the presentation of the data in this manner. So it amounts to academic
catfishing, in my opinion, to even suggest that.

Now let’s take a look at mortality *rates* over time broken out by the
age cohorts.

    plot3.data <- data2 %>% 
      group_by(Year, 
               Age_Group_Regroup = if_else(Cohort %in% c("< 1 year","1-4 years","5-14 years"), "14 and under", Cohort)) %>%
      summarize(Deaths = sum(Deaths, na.rm=T),
                Population = sum(Population, na.rm=T)) %>% 
      mutate(TotalPop = sum(Population, na.rm=T)) %>% 
      ungroup() %>% 
      mutate(q_x = Deaths / Population,
             contrib_to_q_x = Deaths / TotalPop)

    ## `summarise()` regrouping output by 'Year' (override with `.groups` argument)

    plot3a <- plot3.data %>% 
      ggplot(mapping=aes(x=Year, y=q_x,
                         color=fct_rev(Age_Group_Regroup))) +
      geom_line() + 
      theme_minimal() +
      scale_color_viridis_d() +
      scale_y_continuous(label=scales::percent) +
      scale_x_continuous(breaks=seq(1999, 2018, 1)) +
      theme(axis.text.x = element_text(angle=90)) +
      labs(x="Year",
           y="Mortality Rate from All Causes",
           title="Observed US Mortality Rates by Age Cohort and Year",
           caption="Source: CDC",
           color="Age Cohort") 
      
    plot3a

![](README_files/figure-markdown_strict/mortality-over-time-2-1.png)
What a lovely sight! Mortality *rates* appear to be going down for the
older ages. The compression of the mortality curves down at the bottom
conceals a bit of a tragedy we’ll unearth in a moment, but we can see
that, in general, it’s never been a better time to be elderly in
America.

Here’s another view similar to the first to graphs, where we’ll break
down each age cohort’s contribution to the total mortality rate in the
US for that year.

    plot3b <- plot3.data %>% 
      ggplot(mapping=aes(x=as_factor(Year), y=contrib_to_q_x,
                         fill=fct_rev(Age_Group_Regroup))) +
      geom_bar(position="stack", stat="identity", color="black", size=0.3) +
      theme_minimal() +
      scale_fill_viridis_d() +
      scale_y_continuous(label=scales::percent) +
      scale_x_discrete(breaks=seq(1999, 2018, 1)) +
      theme(axis.text.x = element_text(angle=90)) +
      labs(x="Year",
           title="Observed Deaths as a Percentage of Total Population",
           y="Observed % of Total Population",
           caption="Source: CDC",
           fill="Age Cohort") 
           
    plot3b

![](README_files/figure-markdown_strict/mortality-over-time-3-1.png)

This is interesting because it tells a slightly different story than the
prior graph - that expressed as a percentage of the population, deaths
*in aggregate* are on the rise in the last decade or so in the US.

This is unsurprising: we see that the largest percentages of death are
focused into the elder age bands, but we already know that these age
bands are seeing decreased mortality rates. In fact, we can see in this
graph more clearly the substantial increase in the size of the middle
age bands, confirming what we know to be true about the toll of the
Opioid Crisis in the US over the last decade.

## Putting it all together

One thing I’d like to do now is combine the historical data I’ve
obtained with the 2020 COVID dataset, but the latter does not have a
population total tabulation by age. Because I am holding myself to
building visuals using *only* actual available CDC data, I will instead
just plot raw deaths over time, from 1999 - 2018, and 2020, all on the
same graph.

We have a problem, though. Our CDC data is really complete through, at
best, 10/31/2020, and it is also missing January 2020! So I’m going to
have to go back to CDC Wonder to grab total historical death figures by
month and year instead of just by month. I’ve got the monthly data
pulled in the file `Underlying Cause of Death, 1999-2018_pull2.txt` and
cut out January, November, and December for each year so that we’re
dealing with a genuine apples-to-apples comparison.

Because I think we’ve thrashed this dead horse, and because I like to
end on a “cut the crap” visualization that shows things as they are, no
filtration, no rates, just raw numbers and sizes and colors: I present
to you a stacked histogram of raw deaths over time in the US.

    plot4.data.historical <- read_delim("Underlying Cause of Death, 1999-2018_pull2.txt", 
        "\t", escape_double = FALSE, col_types = cols(Notes = col_skip(), 
            `Ten-Year Age Groups Code` = col_skip(), 
            Month = col_skip(), Population = col_skip(), 
            `Crude Rate` = col_skip()), trim_ws = TRUE) %>% 
      filter(as.numeric(str_sub(`Month Code`, 6)) <= 10 &
               as.numeric(str_sub(`Month Code`, 6)) >= 2) %>% 
      group_by(Year=as.numeric(str_sub(`Month Code`, 1,4)), 
               Age_Group_Regroup =
                 if_else(`Ten-Year Age Groups` %in% c("< 1 year","1-4 years","5-14 years"),
                         "14 and under", `Ten-Year Age Groups`)) %>% 
      summarize(Deaths = sum(Deaths, na.rm=T)) %>% 
      ungroup() %>% 
      mutate(`COVID-19 Deaths` = 0) %>% 
      arrange()

    ## Warning: 29 parsing failures.
    ##  row col  expected    actual                                             file
    ## 2653  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018_pull2.txt'
    ## 2654  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018_pull2.txt'
    ## 2655  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018_pull2.txt'
    ## 2656  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018_pull2.txt'
    ## 2657  -- 8 columns 1 columns 'Underlying Cause of Death, 1999-2018_pull2.txt'
    ## .... ... ......... ......... ................................................
    ## See problems(...) for more details.

    ## `summarise()` regrouping output by 'Year' (override with `.groups` argument)

    plot4.data <- final_table %>% 
      ungroup() %>% 
      filter(Sex == "All Sex",
             `Age Group` != "All Ages",
             `End Week` <= ymd(20201031)) %>% 
      group_by(Year=year(`End Week`), 
               Age_Group_Regroup =
                 if_else(`Age Group` %in% c("Under 1 year","1-4 years","5-14 years"),
                         "14 and under", `Age Group`)) %>% 
      summarize(`Deaths` = sum(`Total Deaths`, na.rm=T),
                `COVID-19 Deaths` = sum(`COVID-19 Deaths`, na.rm=T)) %>% 
      ungroup() %>% 
      mutate(Age_Group_Regroup = if_else(Age_Group_Regroup == "85 years and over", 
                                         "85+ years",Age_Group_Regroup)) %>% 
      union_all(plot4.data.historical)

    ## `summarise()` regrouping output by 'Year' (override with `.groups` argument)

    plot4 <- plot4.data %>% 
      ggplot(mapping=aes(x=as_factor(Year), y=Deaths,
                         fill=fct_rev(Age_Group_Regroup))) +
      geom_bar(position="stack", stat="identity", color="black", size=0.3) +
      theme_minimal() +
      scale_fill_viridis_d() +
      scale_y_continuous(label=scales::comma_format()) +
      scale_x_discrete(breaks=seq(1999, 2020, 1)) +
      theme(axis.text.x = element_text(angle=90)) +
      labs(x="Year",
           y="Observed Deaths",
           title="Total Deaths by Year and Age Banded Cohort",
           caption="Source: CDC",
           fill="Age Cohort") 

    plot4

![](README_files/figure-markdown_strict/mortality-over-time-4-1.png)

That’s a pretty clear difference.

> Briand also noted that 50,000 to 70,000 deaths are seen both before
> and after COVID-19, indicating that this number of deaths was normal
> long before COVID-19 emerged. Therefore, according to Briand, not only
> has COVID-19 had no effect on the percentage of deaths of older
> people, but it has also not increased the total number of deaths.

> These data analyses suggest that in contrast to most people’s
> assumptions, the number of deaths by COVID-19 is not alarming. In
> fact, it has relatively no effect on deaths in the United States.

I mean this with all due professionalism: **GTFOH.**

## Cherry picking data is @\#$%ing unethical

This is a bonus section - I want to scrutinize an exhibit Briand put
together about reporting discrepancies. This image:

![Reporting
Discrepancies](https://web.archive.org/web/20201126181148im_/https://snworksceo.imgix.net/jhn/6b057424-a047-49bd-96b5-c0a65cbce88a.sized-1000x1000.png?w=2000&dpr=1.5)

To go along with this exhibit we have this juicy nugget:

> This trend is completely contrary to the pattern observed in all
> previous years. Interestingly, as depicted in the table \[above\], the
> total decrease in deaths by other causes almost exactly equals the
> increase in deaths by COVID-19. This suggests, according to Briand,
> that the COVID-19 death toll is misleading. Briand believes that
> deaths due to heart diseases, respiratory diseases, influenza and
> pneumonia may instead be recategorized as being due to COVID-19.

Then the icing on the cake is the historical pattern of deaths plotted
by cause for the last 5 years, and inset on that graph is a depiction of
this small sliver of 6 weeks in which COVID data is off the charts.

![COVID
Misreporting?](https://web.archive.org/web/20201126181148im_/https://snworksceo.imgix.net/jhn/943c93ab-5e5e-4402-a235-5e756030ca8f.sized-1000x1000.png?w=2000&dpr=1.5)

Let’s re-create this chart. Because this is puzzling, for sure, but if
I’m going to take this argument seriously I would want to see continued
evidence of this sustained throughout 2020 and not just in the opening
months of the pandemic.

Isn’t it plausible to believe in the early days of the pandemic, when we
were first figuring out which end is up, how testing could be done
reliably, and getting hospital systems worked out for the long haul -
that some cases were misreported as other things? I believe so, and I
don’t think that’s malicious, and I don’t think Briand would think that
either.

But to build a whole case on there being systemic misreported COVID data
based on three weeks of data is a risky move, and I’m going to call her
on it.

So, I’ll once again retrieve her same *exact* data and prepare the same
*exact* exhibit, for validation purposes. And then I will extend the
graph to all of 2020 data available on the CDC data portal in order to
see if the COVID reporting phenomenon she’s describing persists
throughout 2020, or if she’s blowing smoke up our collective asses.
Because if it’s the latter, and I say this with all due force of
professionalism, **she should stick to running a master’s program in
Economics at the Hop and stay out of mortality analysis**.

    library(ISOweek)

    url <- "https://data.cdc.gov/api/views/u6jv-9ijr/rows.json"
    params <- list(accessType="DOWNLOAD")

    get <- GET(url = url, config=params)
    print(paste0("That worked OK? ", !http_error(get)))

    ## [1] "That worked OK? TRUE"

    data <- fromJSON(content(get, as="text"))

    table2 <- as_tibble(data$data)
    names(table2) <- data[["meta"]][["view"]][["columns"]][["name"]]

    final_table2 <- table2 %>% 
      select(9:23) %>% 
      mutate(`Number of Deaths` = as.numeric(`Number of Deaths`),
             `Week Ending Date` = as.Date(`Week Ending Date`),
             isow =   as.character(ISOweek::date2ISOweek(`Week Ending Date`)),
             pretty_isoweek = str_sub(isow, 6, 8)) %>% 
      filter(Type == "Unweighted",
             Jurisdiction == "United States",
             `Week Ending Date` <= ymd(20201031),
             Year >= 2016) %>% 
      group_by(pretty_isoweek, Year, `Cause Group`) %>% 
      summarize(`Number of Deaths` = sum(`Number of Deaths`, na.rm=T)) %>% 
      ungroup() %>% 
      arrange(pretty_isoweek, Year)

    ## `summarise()` regrouping output by 'pretty_isoweek', 'Year' (override with `.groups` argument)

    plot5 <- final_table2 %>% 
      ggplot(aes(x = as_factor(pretty_isoweek), y = `Number of Deaths`,
                 color=Year, group=Year)) +
      #geom_bar(stat="identity", position="stack", color="black", size=0.01) + 
      geom_line() +
      theme_minimal() +
      scale_color_viridis_d() +
      scale_y_continuous(label=scales::comma_format(),
                         limits=c(0,10000)) +
      scale_x_discrete() +
      theme(axis.text.x = element_text(angle=90),
            legend.position = "bottom") +
      labs(x="Year",
           y="Observed Deaths",
           title="Total Deaths by Year and Age Banded Cohort",
           caption="Source: CDC",
           fill="Age Cohort") +
      facet_wrap(facets= `Cause Group`~., ncol=4)

    plot5

![](README_files/figure-markdown_strict/fuck-cherry-pickers-1.png)

– This is an R Markdown document. Markdown is a simple formatting syntax
for authoring HTML, PDF, and MS Word documents. For more details on
using R Markdown see <http://rmarkdown.rstudio.com>.
