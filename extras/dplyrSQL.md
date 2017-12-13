dplyrSQL
================
Win-Vector LLC
12/11/2017

`dplyr` SQL for the [`rquery` example](https://winvector.github.io/rquery/). Notice the irrelevant columns live a few steps into the query sequence. Also notice the `dplyr` `SQL` does have less nesting than the `rquery` `SQL`.

``` r
suppressPackageStartupMessages(library("dplyr"))
packageVersion("dplyr")
```

    ## [1] '0.7.4'

``` r
my_db <- sparklyr::spark_connect(version='2.2.0', 
                                 master = "local")
```

    ## Warning in yaml.load(readLines(con), error.label = error.label, ...): R
    ## expressions in yaml.load will not be auto-evaluated by default in the near
    ## future

    ## Warning in yaml.load(readLines(con), error.label = error.label, ...): R
    ## expressions in yaml.load will not be auto-evaluated by default in the near
    ## future

    ## Warning in yaml.load(readLines(con), error.label = error.label, ...): R
    ## expressions in yaml.load will not be auto-evaluated by default in the near
    ## future

``` r
d <- dplyr::copy_to(my_db,
                    data.frame(
                      subjectID = c(1,
                                    1,
                                    2,
                                    2),
                      surveyCategory = c(
                        'withdrawal behavior',
                        'positive re-framing',
                        'withdrawal behavior',
                        'positive re-framing'
                      ),
                      assessmentTotal = c(5,
                                          2,
                                          3,
                                          4),
                      irrelevantCol1 = "irrel1",
                      irrelevantCol2 = "irrel2",
                      stringsAsFactors = FALSE),
                    name =  'd',
                    temporary = TRUE,
                    overwrite = FALSE)

scale <- 0.237

dq <- d %>%
  group_by(subjectID) %>%
  mutate(probability =
           exp(assessmentTotal * scale)/
           sum(exp(assessmentTotal * scale))) %>%
  arrange(probability, surveyCategory) %>%
  mutate(isDiagnosis = row_number() == n()) %>%
  filter(isDiagnosis) %>%
  ungroup() %>%
  select(subjectID, surveyCategory, probability) %>%
  rename(diagnosis = surveyCategory) %>%
  arrange(subjectID)

# directly prints, can not easilly and reliable capture SQL
show_query(dq)
```

    ## <SQL>
    ## SELECT `subjectID` AS `subjectID`, `surveyCategory` AS `diagnosis`, `probability` AS `probability`
    ## FROM (SELECT `subjectID` AS `subjectID`, `surveyCategory` AS `surveyCategory`, `probability` AS `probability`
    ## FROM (SELECT *
    ## FROM (SELECT `subjectID`, `surveyCategory`, `assessmentTotal`, `irrelevantCol1`, `irrelevantCol2`, `probability`, row_number() OVER (PARTITION BY `subjectID` ORDER BY `probability`, `surveyCategory`) = COUNT(*) OVER (PARTITION BY `subjectID`) AS `isDiagnosis`
    ## FROM (SELECT *
    ## FROM (SELECT `subjectID`, `surveyCategory`, `assessmentTotal`, `irrelevantCol1`, `irrelevantCol2`, EXP(`assessmentTotal` * 0.237) / sum(EXP(`assessmentTotal` * 0.237)) OVER (PARTITION BY `subjectID`) AS `probability`
    ## FROM `d`) `btiutqisxf`
    ## ORDER BY `probability`, `surveyCategory`) `ohdcwqtxwh`) `odjvjxiujp`
    ## WHERE (`isDiagnosis`)) `sqscykgqmy`) `xdedicvrse`
    ## ORDER BY `subjectID`

``` r
# directly prints, can not easilly and reliable capture SQL
explain(dq)
```

    ## <SQL>
    ## SELECT `subjectID` AS `subjectID`, `surveyCategory` AS `diagnosis`, `probability` AS `probability`
    ## FROM (SELECT `subjectID` AS `subjectID`, `surveyCategory` AS `surveyCategory`, `probability` AS `probability`
    ## FROM (SELECT *
    ## FROM (SELECT `subjectID`, `surveyCategory`, `assessmentTotal`, `irrelevantCol1`, `irrelevantCol2`, `probability`, row_number() OVER (PARTITION BY `subjectID` ORDER BY `probability`, `surveyCategory`) = COUNT(*) OVER (PARTITION BY `subjectID`) AS `isDiagnosis`
    ## FROM (SELECT *
    ## FROM (SELECT `subjectID`, `surveyCategory`, `assessmentTotal`, `irrelevantCol1`, `irrelevantCol2`, EXP(`assessmentTotal` * 0.237) / sum(EXP(`assessmentTotal` * 0.237)) OVER (PARTITION BY `subjectID`) AS `probability`
    ## FROM `d`) `msneohpzga`
    ## ORDER BY `probability`, `surveyCategory`) `dcixpqdkmw`) `czqosedexy`
    ## WHERE (`isDiagnosis`)) `htefgrxelx`) `lrxpmtccvs`
    ## ORDER BY `subjectID`

    ## 

    ## <PLAN>

``` r
dq
```

    ## # Source: lazy query [?? x 3]
    ## # Database: spark_connection
    ## # Ordered by: probability, surveyCategory, subjectID
    ##   subjectID diagnosis           probability
    ##       <dbl> <chr>                     <dbl>
    ## 1      1.00 withdrawal behavior       0.671
    ## 2      2.00 positive re-framing       0.559

``` r
sparklyr::spark_disconnect(my_db)
```