# nzelect and nzcensus
New Zealand election results, polling data and census results data in convenient form of two R packages.  Each of the two packages can be installed separately, but they have been developed together and get good results working together.

[![Travis-CI Build Status](https://travis-ci.org/ellisp/nzelect.svg?branch=master)](https://travis-ci.org/ellisp/nzelect)

## Installation
`nzelect` is on CRAN, but `nzcensus` is too large so will remain on GitHub only.

```r
# install stable version of nzelect from CRAN:
install.packages("nzelect")

# or install dev version of nzelect (with the very latest data) from GitHub:
devtools::install_github("ellisp/nzelect/pkg1")


# install nzcensus from GitHub:
devtools::install_github("ellisp/nzelect/pkg2")

library(nzelect)
library(nzcensus)
```


# nzelect

[![CRAN version](http://www.r-pkg.org/badges/version/nzelect)](http://www.r-pkg.org/pkg/nzelect)
[![CRAN RStudio mirror downloads](http://cranlogs.r-pkg.org/badges/nzelect)](http://www.r-pkg.org/pkg/nzelect)

## Caveat and disclaimer

The New Zealand Electoral Commission had no involvement in preparing this package and bear no responsibility for any errors.  In the event of any uncertainty, refer to the definitive source materials on their website.

`nzelect` is a very small voluntary project.  Please report any issues or bugs on [GitHub](https://github.com/ellisp/nzelect/issues).

## Usage - 2002 to 2014 results by voting place

The election results are available in two main data frames:

* `voting_places` has one row for each of election year - voting place combination
* `nzge` has one row for each combination of election year, voting place, party, electorate and voting type (Party or Candidate)

### Overall results
The code below replicates the published results for the 2011 election at http://www.electionresults.govt.nz/electionresults_2011/e9/html/e9_part1.html

```r
library(nzelect)
library(tidyr)
library(dplyr)
nzge %>%
    filter(election_year == 2011) %>%
    mutate(voting_type = paste0(voting_type, " Vote")) %>%
    group_by(party, voting_type) %>%
    summarise(votes = sum(votes)) %>%
    spread(voting_type, votes) %>%
    ungroup() %>%
    arrange(desc(`Party Vote`))
```

```
## # A tibble: 25 x 3
##                      party `Candidate Vote` `Party Vote`
##                      <chr>            <dbl>        <dbl>
##  1          National Party          1027696      1058636
##  2            Labour Party           762897       614937
##  3             Green Party           155492       247372
##  4 New Zealand First Party            39892       147544
##  5      Conservative Party            51678        59237
##  6             Maori Party            39320        31982
##  7                    Mana            29872        24168
##  8         ACT New Zealand            31001        23889
##  9    Informal Party Votes               NA        19872
## 10           United Future            18792        13443
## # ... with 15 more rows
```


### Comparing party and candidate votes of several parties

```r
library(ggplot2, quietly = TRUE)
library(scales, quietly = TRUE)
library(GGally, quietly = TRUE) # for ggpairs
library(dplyr)

proportions <- nzge %>%
    filter(election_year == 2014) %>%
    group_by(voting_place, voting_type) %>%
    summarise(`proportion Labour` = sum(votes[party == "Labour Party"]) / sum(votes),
              `proportion National` = sum(votes[party == "National Party"]) / sum(votes),
              `proportion Greens` = sum(votes[party == "Green Party"]) / sum(votes),
              `proportion NZF` = sum(votes[party == "New Zealand First Party"]) / sum(votes),
              `proportion Maori` = sum(votes[party == "Maori Party"]) / sum(votes))

ggpairs(proportions, aes(colour = voting_type), columns = 3:5)
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3-1.png)



### Geographical location of voting places

These are most reliable and checked for 2014.  Please raise an issue on GitHub if you spot anything.


```r
library(ggthemes) # for theme_map()
nzge %>%
    filter(voting_type == "Party" & election_year == 2014) %>%
    group_by(voting_place, election_year) %>%
    summarise(proportion_national = sum(votes[party == "National Party"] / sum(votes))) %>%
    left_join(voting_places, by = c("voting_place", "election_year")) %>%
    filter(voting_place_suburb != "Chatham Islands") %>%
    mutate(mostly_national = ifelse(proportion_national > 0.5, 
                                   "Mostly voted National", "Mostly didn't vote National")) %>%
    ggplot(aes(x = longitude, y = latitude, colour = proportion_national)) +
    geom_point() +
    facet_wrap(~mostly_national) +
    coord_map() +
    borders("nz") +
    scale_colour_gradient2(label = percent, mid = "grey80", midpoint = 0.5) +
    theme_map() +
    theme(legend.position = c(0.04, 0.5)) +
    ggtitle("Voting patterns in the 2014 General Election\n")
```

```
## Warning: `panel.margin` is deprecated. Please use `panel.spacing` property
## instead
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4-1.png)

See this [detailed interactive map of of the 2014 general election](https://ellisp.shinyapps.io/NZ-general-election-2014/) 
built as a side product of this project.

### Rolling up results to Regional Council, Territorial Authority, or Area Unit
Because this package matches the location people actually voted with to boundaries 
of Regional Council, Territorial Authority and Area Unit it's possible to roll up 
voting behaviour to those categories.  However, a large number of votes cannot be
located this way.  And it needs to be remembered that people are not necessarily voting
near their normal place of residence.

```r
nzge %>%
    filter(election_year == 2014) %>%
    filter(voting_type == "Party") %>%
    left_join(voting_places, by = c("voting_place", "election_year")) %>%
    group_by(REGC2014_N) %>%
    summarise(
        total_votes = sum(votes),
        proportion_national = round(sum(votes[party == "National Party"]) / total_votes, 3)) %>%
    arrange(proportion_national)
```

```
## # A tibble: 17 x 3
##                  REGC2014_N total_votes proportion_national
##                      <fctr>       <dbl>               <dbl>
##  1          Gisborne Region       14342               0.351
##  2            Nelson Region       18754               0.398
##  3         Northland Region       53688               0.427
##  4        Wellington Region      164913               0.430
##  5 Manawatu-Wanganui Region       78841               0.447
##  6             Otago Region       75933               0.447
##  7                     <NA>      935376               0.451
##  8       Hawke's Bay Region       53833               0.460
##  9            Tasman Region       17935               0.465
## 10        West Coast Region       12226               0.465
## 11     Bay of Plenty Region       89065               0.473
## 12          Auckland Region      478724               0.486
## 13           Waikato Region      134511               0.512
## 14        Canterbury Region      192120               0.520
## 15       Marlborough Region       17474               0.520
## 16         Southland Region       36158               0.528
## 17          Taranaki Region       42586               0.552
```

```r
# what are all those NA Regions?:
nzge %>%
    filter(voting_type == "Party" & election_year == 2014) %>%
    left_join(voting_places, by = c("voting_place", "election_year")) %>%
    filter(is.na(REGC2014_N)) %>%
    group_by(voting_place) %>%
    summarise(total_votes = sum(votes))
```

```
## # A tibble: 10 x 2
##                                               voting_place total_votes
##                                                      <chr>       <dbl>
##  1 Chatham Islands Council Building, 9 Tuku Road, Waitangi          90
##  2  Mount Pleasant Community Centre, 3 Mccormacks Bay Road         457
##  3                       Ordinary Votes BEFORE polling day      630775
##  4               Otaki Surf Lifesaving Club, Marine Parade         294
##  5          Overseas Special Votes including Defence Force       38316
##  6              Port Fitzroy Aotea Centre (Nurses Cottage)          36
##  7                        Special Votes BEFORE polling day       71362
##  8                            Special Votes On polling day      151530
##  9                            Votes Allowed for Party Only       40986
## 10        Voting places where less than 6 votes were taken        1530
```

```r
nzge %>%
    filter(voting_type == "Party" & election_year == 2014) %>%
    left_join(voting_places, by = c("voting_place", "election_year")) %>%
    group_by(TA2014_NAM) %>%
    summarise(
        total_votes = sum(votes),
        proportion_national = round(sum(votes[party == "National Party"]) / total_votes, 3)) %>%
    arrange(desc(proportion_national)) %>%
    mutate(TA = ifelse(is.na(TA2014_NAM), "Special or other", as.character(TA2014_NAM)),
           TA = gsub(" District", "", TA),
           TA = gsub(" City", "", TA),
           TA = factor(TA, levels = TA)) %>%
    ggplot(aes(x = proportion_national, y = TA, size = total_votes)) +
    geom_point() +
    scale_x_continuous("Proportion voting National Party", label = percent) +
    scale_size("Number of\nvotes cast", label = comma) +
    labs(y = "", title = "Voting in the New Zealand 2014 General Election by Territorial Authority")
```

![plot of chunk unnamed-chunk-5](figure/unnamed-chunk-5-1.png)

## Usage - Opinion polls

Opinion poll data from 2002 onwards has been tidied and collated into a single data object, `polls`.  Note that at the time of writing, sample sizes are not yet available.  The example below illustrates use of the few years of polling data since the 2014 election, in conjunction with the `parties_v` object which provides colours to use in representing political parties in graphics.


```r
library(forcats)
polls %>%
    filter(MidDate > as.Date("2014-11-20") & !is.na(VotingIntention)) %>%
    filter(Party %in% c("National", "Labour", "Green", "NZ First")) %>%
    mutate(Party = fct_reorder(Party, VotingIntention, .desc = TRUE),
           Party = fct_drop(Party)) %>%
    ggplot(aes(x = MidDate, y = VotingIntention, colour = Party, linetype = Pollster)) +
    geom_line(alpha = 0.5) +
    geom_point(aes(shape = Pollster)) +
    geom_smooth(aes(group = Party), se = FALSE, colour = "grey15", span = .4) +
    scale_colour_manual(values = parties_v) +
    scale_y_continuous("Voting intention", label = percent) +
    scale_x_date("") +
    facet_wrap(~Party, scales = "free_y") 
```

```
## `geom_smooth()` using method = 'loess'
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6-1.png)

Note that it is not appropriate to frequently update the version of `nzelect` on CRAN, so polling data will generally be out of date.  The development version of `nzelect` from GitHub will be kept more up to date (but no promises exactly how much).

## Usage - convenience functions

### Allocating parliamentary seats
The `allocate_seats` function uses the Sainte-Lague allocation method to allocate seats to a Parliament given proportions or counts of vote per party.  When used with the default settings, it should give the same result as the New Zealand Electoral Commission; this means a five percent threshold to be included in the main algorithm, but parties below five percent of total votes but with at least one electorate seat get total seats proportionate to their votes.  Here is the `allocate_seats` function in action with the actual vote counts from the 2014 General Election:


```r
votes <- c(National = 1131501, Labour = 604535, Green = 257359,
           NZFirst = 208300, Cons = 95598, IntMana = 34094, 
           Maori = 31849, Act = 16689, United = 5286,
           Other = 20411)
electorate = c(41, 27, 0, 
               0, 0, 0, 
               1, 1, 1,
               0)
               
# Actual result:               
allocate_seats(votes, electorate = electorate)
```

```
## $seats_df
##    proportionally_allocated electorate_seats final    party
## 1                        60               41    60 National
## 2                        32               27    32   Labour
## 3                        14                0    14    Green
## 4                        11                0    11  NZFirst
## 5                         0                0     0     Cons
## 6                         0                0     0  IntMana
## 7                         2                1     2    Maori
## 8                         1                1     1      Act
## 9                         0                1     1   United
## 10                        0                0     0    Other
## 
## $seats_v
## National   Labour    Green  NZFirst     Cons  IntMana    Maori      Act 
##       60       32       14       11        0        0        2        1 
##   United    Other 
##        1        0
```

```r
# Result if there were no 5% minimum threshold:
allocate_seats(votes, electorate = electorate, threshold = 0)$seats_v
```

```
## National   Labour    Green  NZFirst     Cons  IntMana    Maori      Act 
##       56       30       13       10        5        2        2        1 
##   United    Other 
##        1        1
```

### Weighting opinion polls

Two techniques are provided in the `weight_polls` function for aggregating opinion polls while giving more weight to more recent polls.  These methods aim to replicate the approaches of the [Pundit Poll of Polls](http://www.pundit.co.nz/content/poll-of-polls), which states it is based on FiveThirtyEight's method; and the [curia Market Research Public Poll Average](http://www.curia.co.nz/).  To date, exact replication of Pundit or curia's results has not been possible, probably due in part to the non-inclusion of sample size data so far in the `polls` data in `nzelect` package.

The example below shows the `weight_polls` function in action in combination with `allocate_seats`, comparing the outcomes of both methods of polling aggregation, on assumption that electorate seats stay as they are in early 2017 (in particular, that ACT, United Future, and Maori party all win at least one electorate seat as needed to keep them in running for the proportional representation part of the seat allocation process).

```r
# electorate seats for Act, Cons, Green, Labour, Mana, Maori, National, NZFirst, United,
# assuming that electorates stay as currently allocated.  This is critical particularly
# for ACT, Maori and United Future, who if they lose their single electorate seat each
# will not be represented in parliament
electorates <- c(1,0,0,27,0,1,41,1,1)

polls %>%
    filter(MidDate > "2014-12-30" & MidDate < "2017-10-1" & Party != "TOP") %>%
    mutate(wt_p = weight_polls(MidDate, method = "pundit", refdate = as.Date("2017-09-22")),
           wt_c = weight_polls(MidDate, method = "curia", refdate = as.Date("2017-09-22"))) %>%
    group_by(Party) %>%
    summarise(pundit_perc = round(sum(VotingIntention * wt_p, na.rm = TRUE) / sum(wt_p) * 100, 1),
              curia_perc = round(sum(VotingIntention * wt_c, na.rm = TRUE) / sum(wt_c) * 100, 1)) %>%
    ungroup() %>%
    mutate(pundit_seats = allocate_seats(pundit_perc, electorate = electorates)$seats_v,
           curia_seats = allocate_seats(curia_perc, electorate = electorates)$seats_v)
```

```
## # A tibble: 9 x 5
##           Party pundit_perc curia_perc pundit_seats curia_seats
##           <chr>       <dbl>      <dbl>        <dbl>       <dbl>
## 1           ACT         0.5        0.6            1           1
## 2  Conservative         0.3        0.4            0           0
## 3         Green         7.1        6.5            9           8
## 4        Labour        39.3       41.0           48          50
## 5          Mana         0.1        0.1            0           0
## 6         Maori         1.2        1.3            1           2
## 7      National        42.3       40.8           52          50
## 8      NZ First         7.0        7.2            9           9
## 9 United Future         0.1        0.1            1           1
```



# nzcensus examples



```r
library(nzcensus)
library(ggrepel)
ggplot(REGC2013, aes(x = PropPubAdmin2013, y = PropPartnered2013, label = REGC2013_N) ) +
    geom_point() +
    geom_text_repel(colour = "steelblue") +
    scale_x_continuous("Proportion of workers in public administration", label = percent) +
    scale_y_continuous("Proportion of individuals who stated status that have partners", label = percent) +
    ggtitle("New Zealand census 2013")
```

![plot of chunk unnamed-chunk-9](figure/unnamed-chunk-9-1.png)


```r
ggplot(Meshblocks2013, aes(x = WGS84Longitude, y = WGS84Latitude, colour = MedianIncome2013)) +
    borders("nz", fill = terrain.colors(5)[3], colour = NA) +
    geom_point(alpha = 0.1) +
    coord_map(xlim = c(166, 179)) +
    theme_map() +
    ggtitle("Locations of centers of meshblocks in 2013 census") +
    scale_colour_gradientn(colours = c("blue", "white", "red"), label = dollar) +
    theme(legend.position = c(0.1, 0.6))
```

```
## Warning: `panel.margin` is deprecated. Please use `panel.spacing` property
## instead
```

```
## Warning: Removed 13 rows containing missing values (geom_point).
```

![plot of chunk unnamed-chunk-10](figure/unnamed-chunk-10-1.png)




# combining nzcensus and nzelect

