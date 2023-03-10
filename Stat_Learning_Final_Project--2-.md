Survival Analysis Tutorial
================
Rory Black, Linnaea Kavulich, Tatev Kyosababyan, Aubrey Winger

### $\textbf{Introduction}$ <br />

This tutorial, will demonstrate how to implement survival analysis to
evaluate the lifetime expectancy of patients with lung cancer. Survival
analysis allows for easy analysis of data that contains information on
the time until an event (death, for instance). Using this technique, we
can represent survival in the form of a probability instead of a
discrete response we might see using other statistical techniques.
<br /> <br /> Also known as reliability or duration analysis, survival
analysis is a statistical method aimed at predicting the amount of time
until a given event occurs. While survival analysis can, as the name
suggests, be employed to anticipate the lifespan of a biological
organism, it has much broader utility – appearing in the fields of
economics, engineering and sociology. <br /> All forms of survival
analysis fundamentally depend on two basic metrics: the survival
function, or the probability that an entity lasts longer than t units of
time, and the hazard function, or the probability that a given entity
dies (or breaks, or an alternate event of interest occurs) given that it
has survived for time t. In its most general form the survival function,
$S(t)$, is written as: $$ S(t) = 1- F(t)$$ Where $F(t)$ is the
distribution function which describes the probability that an entity
persists up to t units of time; the form $F(t)$ adopts depends on the
nature of the entity being studied. The hazard function $\lambda (t)$ is
shown below: <br /> <br />

<center>
$\lambda (t) =\frac{f(t)}{S(t)}$ where $f(t) = \frac{dF(t)}{dt}$
</center>

<br /> With this essential background in place, we may now examine the
various mechanisms of survival analysis. There are three main functions
of survival analysis, each accompanied by unique strategies for
implementation – the most common application being to depict the
survival times among members of a certain group (think of a cohort of
cancer patients). Survival time is almost ubiquitous with a Kaplan-Meier
curve (See section “Kaplan-Meier Method”) in which the probability of an
entity’s survival is modeled as a function of time. In a similar
fashion, survival analysis can be used to compare the survival times
between groups, as is useful when testing novel drugs on rodents. This
inter-group analysis can be attained with a log-rank test, which
essentially estimates and compares the hazard functions of two groups of
entities to assess their relative survival. Finally, it is possible to
use survival analysis techniques to examine the impact of both
quantitative and categorical variables on survival outcomes, the most
common of which is Cox Regression. <br /> <br /> As alluded to, survival
analysis is best employed when the object of interest is the amount of
time until some anticipated event. Ideally, this event would be of an
unambiguous nature (an organism lives or dies, a person is incarcerated,
etc.) as survival analysis is ill-equipped to deal with outcomes which
occur on a spectrum. Survival analysis is particularly well suited to
capturing complex interactions between variables, making it an excellent
choice in addressing problems like the duration of unemployment. Here we
seek to provide a functional understanding of survival analysis methods,
allowing the reader to leverage the unique analytic insight survival
analysis has to offer. <br />

### $\textbf{Load Required Packages}$ <br />

If packages are not already installed, execute
`install.packages("package")` where *package* is the name of the package
that you are attempting to install.

``` r
library(dplyr)     # functions for data manipulation
library(glmnet)    # implement Cox models
library(survival)  # Kaplan-Meier curve
library(survminer) # Kaplan-Meier curve
library(Rcpp)      # log-rank test
library(ggplot2)   # plotting
library(tidyverse) # additional functions for data manipulation
```

### $\textbf{Data Exploration}$ <br />

In this example, we will use a data set from the `survival` package with
information on lung cancer. There are a total of 228 observations and 10
different variables. These variables and their significance are listed
below.

-   `inst`: A code to represent the institution

-   `time`: Survival time (days)

-   `status`: Censoring status indicates if patient died or left study
    in another way; 0=censored, 1=dead

-   `age`: Age of patient (years)

-   `sex`: Sex of patient; 1=Male, 2=Female

-   `ph.ecog`: ECOG performance score; 0=asymptomatic, 1=symptomatic but
    completely ambulatory, 2=in bed \<50% of the day, 3=in bed >50% of
    the day but not bedbound, 4=bedbound

-   `ph.karno`: Karnofsky performance score rated by physician; scale
    from 0 to 100 where 0=bad and 100=good

-   `pat.karno`: Karnofsky performance score rated by patient; scale
    from 0 to 100 where 0=bad and 100=good

-   `meal.cal`: Calories consumed through meals

-   `wt.loss`: Weight lost over the last six months (pounds)

Before proceeding with the analysis, we first need to read in the data
and ensure that it is properly formatted for the analysis. In the
original data, `status` is marked as either a 1 or 2 to represent
“censored” or “dead”. We will re-code this to be either a 0 or 1.The
first two features, `X` (which is an index), and `inst`, were dropped
because they were not relevant to the cancer analysis. Censored patients
are patients who left the study or whose outcome is otherwise unknown.
It is important to keep track of censored patients so that they are not
assumed to survive throughout the study.

``` r
# remove `X` and `inst`
cancer_data = read_csv("cancer.csv")
```

    ## New names:
    ## Rows: 228 Columns: 11
    ## ── Column specification
    ## ──────────────────────────────────────────────────────── Delimiter: "," dbl
    ## (11): ...1, inst, time, status, age, sex, ph.ecog, ph.karno, pat.karno, ...
    ## ℹ Use `spec()` to retrieve the full column specification for this data. ℹ
    ## Specify the column types or set `show_col_types = FALSE` to quiet this message.
    ## • `` -> `...1`

``` r
cancer_data = subset(cancer_data,select=-c(1,2))
# recode `status`
cancer_data <- cancer_data %>% 
  mutate(status=recode(status, `1`=0, `2`=1))

head(cancer_data)                            # display df
```

    ## # A tibble: 6 × 9
    ##    time status   age   sex ph.ecog ph.karno pat.karno meal.cal wt.loss
    ##   <dbl>  <dbl> <dbl> <dbl>   <dbl>    <dbl>     <dbl>    <dbl>   <dbl>
    ## 1   306      1    74     1       1       90       100     1175      NA
    ## 2   455      1    68     1       0       90        90     1225      15
    ## 3  1010      0    56     1       0       90        90       NA      15
    ## 4   210      1    57     1       1       90        60     1150      11
    ## 5   883      1    60     1       0      100        90       NA       0
    ## 6  1022      0    74     1       1       50        80      513       0

``` r
# check for any missing values
na_rows <- rowSums(is.na(cancer_data))      # NA count by row
sum(na_rows != 0)                           # rows with missing values
```

    ## [1] 60

We can notice that there are quite a few rows with missing values. Most
of these missing values are due to either a missing `meal.cal` or
`wt.loss` value. As to not remove this many different observations, we
will continue analysis with the inclusion of missing values. <br />
<br /> We have the ability to proceed with a few different methods to
design a probabilistic approach to evaluating survival time. The
**Kaplan-Meier Method** is method used to visualize survival curves in a
plot format. Building a **Cox Proportional Hazard Regression Model**
allows us to evaluate individual variable effects on the survival
outcome. Lastly, we will utilize the **Log-Rank Test** to view
comparisons of survival across groups.

# $\textbf{Calculating Survival Analysis Manually}$

### $\textbf{Life Table}$ <br />

A life table is used in Survival Analysis to summarize the experiences
of the patients over a period of time. In the case of our dataset, we
are looking at patient deaths from cancer over the interval of the
study. A follow-up life table with probabilities is used widely in
bio-statistical analysis. In order to create a life table, it is
necessary to break the survival time for the patients into intervals.
Since these times range from 5 days to 1022 days, we will consider time
from 0 days to 1030 days, broken up into 5 bins (0-206, 206-412,
412-618, 618-824, 824-1030). These bins are not upper bound inclusive,
and the number of bins was arbitrarily chosen.

``` r
print(min(cancer_data$time))
```

    ## [1] 5

``` r
print((max(cancer_data$time)))
```

    ## [1] 1022

``` r
print(dim(cancer_data))
```

    ## [1] 228   9

The information needed to calculate a life table is the following:
number of patients alive at the beginning of each interval, number of
deaths during each interval, and number of patients censored in each
interval. In order to calculate these, some data filtering had to be
done for each interval.

``` r
int1_a <- 228
int1_d <- as.integer(cancer_data %>%
  filter(time<206&status==1)%>%
  count())
int1_c <- as.integer(cancer_data %>%
  filter(time<206&status==0)%>%
  count())
int2_a <- as.integer(int1_a - (int1_d+int1_c))
int2_d <- as.integer(cancer_data %>%
  filter(time>=206&time<412&status==1)%>%
  count())
int2_c <-as.integer(cancer_data %>%
  filter(time>=206&time<412&status==0)%>%
  count())
int3_a <- as.integer(int2_a - (int2_d + int2_c))
int3_d <- as.integer(cancer_data %>%
  filter(time>=412&time<618&status==1)%>%
  count())
int3_c <- as.integer(cancer_data %>%
  filter(time>=412&time<618&status==0)%>%
  count())
int4_a <- as.integer(int3_a - (int3_d + int3_c))
int4_d <- as.integer(cancer_data %>%
  filter(time>=618&time<824&status==1)%>%
  count())
int4_c <- as.integer(cancer_data %>%
  filter(time>=618&time<824&status==0)%>%
  count())
int5_a <- as.integer(int4_a - (int4_d + int4_c))
int5_d <- as.integer(cancer_data %>%
  filter(time>=824&status==1)%>%
  count())
int5_c <- as.integer(cancer_data %>%
  filter(time>=824&status==0)%>%
  count())
```

Now we have the information we need to calculate the life table.

``` r
life_table <- data.frame(interval_in_days = c("0-205","206-411","412-617","618-823","824-1029"),
  number_alive_at_beginning= c(int1_a,int2_a,int3_a,int4_a,int5_a),
                         num_deaths_during = c(int1_d,int2_d,int3_d,int4_d,int5_d),
                         num_censored = c(int1_c,int2_c,int3_c,int4_c,int5_c))
life_table         
```

    ##   interval_in_days number_alive_at_beginning num_deaths_during num_censored
    ## 1            0-205                       228                75           14
    ## 2          206-411                       139                51           32
    ## 3          412-617                        56                23           10
    ## 4          618-823                        23                15            3
    ## 5         824-1029                         5                 1            4

Life table analysis has a specialized notation; in order to understand
what is being calculated, we will define the notation below. <br />
N$_t$ : Number of patients who are event-free at the start of the
interval, and are at risk during the interval. For our purposes, this
would be the number of patients alive at the beginning of each interval.
<br /> D$_t$ : Number of participants who undergo an event during the
interval. This would be the number of patient deaths during each
interval. <br /> C$_t$ : Number of patients that are censored during
each interval. <br /> N$_t$\* : Average number of patients at risk
during an interval. For a life table this is calculated as
N$_t$-(C$_t$/2), since deaths are assumed to occur at the end of the
interval and censored events are assumed to happen uniformly throughout
the interval. <br /> q$_t$ : D$_t$/N$_t$\* or proportion dying during an
interval. <br /> p$_t$ : 1-q$_t$ or proportion of surviving patients
during an interval (event free). <br /> S$_t$ : Cumulative survival
probability. S$_{t+1}$ = p$_{t+1}$ \* S$_t$. This is defined as the
proportion of patients surviving past a given interval (remaining event
free).

``` r
alive <- c(int1_a,int2_a,int3_a,int4_a,int5_a)
censored <- c(int1_c,int2_c,int3_c,int4_c,int5_c)
death <- c(int1_d,int2_d,int3_d,int4_d,int5_d)
N_t_star <- alive-(censored/2)
q_t <- death/N_t_star
p_t <- 1-q_t
life_table <- data.frame(interval_in_days = c("0-205","206-411","412-617","618-823","824-1029"),
  N_t = alive,
  N_t_star,
  D_t = death,
  C_t = censored,
  q_t = q_t,
  p_t = p_t,
  S_t = c(p_t[1],p_t[1]*p_t[2],p_t[2]*p_t[3],p_t[3]*p_t[4],p_t[4]*p_t[5]))
life_table    
```

    ##   interval_in_days N_t N_t_star D_t C_t       q_t       p_t       S_t
    ## 1            0-205 228    221.0  75  14 0.3393665 0.6606335 0.6606335
    ## 2          206-411 139    123.0  51  32 0.4146341 0.5853659 0.3867123
    ## 3          412-617  56     51.0  23  10 0.4509804 0.5490196 0.3213773
    ## 4          618-823  23     21.5  15   3 0.6976744 0.3023256 0.1659827
    ## 5         824-1029   5      3.0   1   4 0.3333333 0.6666667 0.2015504

Our follow-up life table is now complete. <br />

### $\textbf{Kaplan-Meier Method}$ <br />

This approach addresses the main issue with the life table method, which
is that the survival probabilities change depending on how the intervals
are defined. This is especially relevant when the sample sizes are
small. The Kaplan-Meier approach re-estimates the survival probability
every time an event (death) occurs. First, the `status` column is
grouped by time, and counts are calculated for each time entry for
deaths and censored patients. Then, the dataframe is pivoted so that the
columns are number of deaths (D<sub>t</sub>) and number censored
(C<sub>t</sub>) for each time in days. The number at risk column is
generated by subtracting the number of deaths and number censored from
all previous days from the total number of patients (228). Finally, the
survival probability, which is what we are ultimately trying to
calculate from this table, is calculated with the following formula:
$$S_{t+1} = S_t * \frac{N_{t+1}-D_{t+1}}{N_{t+1}}$$

``` r
cancer_data <- cancer_data[order(cancer_data$time),]
kaplan_meier <- cancer_data %>% group_by(time, status) %>% summarise(total_count=n(),.groups='drop') %>% as.data.frame()
kaplan_meier <- pivot_wider(kaplan_meier,names_from=status,values_from=total_count)
colnames(kaplan_meier) <- c('time_days','num_deaths','num_censored')
kaplan_meier[is.na(kaplan_meier)] <- 0
kaplan_meier$number_risk <- rep(228,nrow(kaplan_meier))
for(i in 1:(nrow(kaplan_meier)-1)){
  kaplan_meier$number_risk[i+1] = kaplan_meier$number_risk[i] - (kaplan_meier$num_deaths[i]+kaplan_meier$num_censored[i]) 
}
kaplan_meier$survival_prob <- rep((1*((228-1)/228)),nrow(kaplan_meier))
for(i in 1:(nrow(kaplan_meier)-1)){
  kaplan_meier$survival_prob[i+1] = kaplan_meier$survival_prob[i]*((kaplan_meier$number_risk[i+1]-kaplan_meier$num_deaths[i+1])/kaplan_meier$number_risk[i+1]) 
}
kaplan_meier
```

    ## # A tibble: 186 × 5
    ##    time_days num_deaths num_censored number_risk survival_prob
    ##        <dbl>      <int>        <int>       <dbl>         <dbl>
    ##  1         5          1            0         228         0.996
    ##  2        11          3            0         227         0.982
    ##  3        12          1            0         224         0.978
    ##  4        13          2            0         223         0.969
    ##  5        15          1            0         221         0.965
    ##  6        26          1            0         220         0.961
    ##  7        30          1            0         219         0.956
    ##  8        31          1            0         218         0.952
    ##  9        53          2            0         217         0.943
    ## 10        54          1            0         215         0.939
    ## # … with 176 more rows

The final Kaplan Meier Approach Table can be seen above! In order to
visualize the changes in survival probability, we will now graph it in
relation to time in days. Notice how it starts arond one, because
patients have a high probability of surviving at the beginning of the
study. As time increases, this probability decreases.

``` r
ggplot(kaplan_meier,aes(time_days,survival_prob))+
  geom_step()+
  xlab("Time in Days")+
  ylab("Survival Probability")+
  ggtitle("Survival Probability vs Time in Days for Lung Cancer Patients")
```

![](Stat_Learning_Final_Project--2-_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

<br /> The survival probability estimated above is a point estimate.
However, we can calculate the standard errors and confidence interval
estimates for these survival probabilities to understand the degree of
uncertainty that accompanies these calculations. The standard error of
survival estimates is calculated with Greenwoods formula:
$$SE(S_t) = S_t \sqrt{ \sum \frac{D_t}{N * (N_t-D_t)}}$$ Let’s add
standard error to our table, and rename it the Standard Errors of
Surival Estimates. We will also add the margin of error (standard error
\* 1.96), which will be useful for calculating the 95% confidence
interval for the survival probability estimates.

``` r
standard_errors <- kaplan_meier
standard_errors <- standard_errors %>% mutate(standard_error = survival_prob*sqrt(cumsum(num_deaths/(number_risk*(number_risk-num_deaths)))))
standard_errors <- standard_errors %>% mutate(margin_of_error=1.96*standard_error)
standard_errors
```

    ## # A tibble: 186 × 7
    ##    time_days num_deaths num_censored number_risk survival_prob standar…¹ margi…²
    ##        <dbl>      <int>        <int>       <dbl>         <dbl>     <dbl>   <dbl>
    ##  1         5          1            0         228         0.996   0.00438 0.00858
    ##  2        11          3            0         227         0.982   0.00869 0.0170 
    ##  3        12          1            0         224         0.978   0.00970 0.0190 
    ##  4        13          2            0         223         0.969   0.0114  0.0224 
    ##  5        15          1            0         221         0.965   0.0122  0.0239 
    ##  6        26          1            0         220         0.961   0.0129  0.0253 
    ##  7        30          1            0         219         0.956   0.0136  0.0266 
    ##  8        31          1            0         218         0.952   0.0142  0.0278 
    ##  9        53          2            0         217         0.943   0.0154  0.0301 
    ## 10        54          1            0         215         0.939   0.0159  0.0312 
    ## # … with 176 more rows, and abbreviated variable names ¹​standard_error,
    ## #   ²​margin_of_error

``` r
ggplot(standard_errors,aes(time_days,survival_prob))+
  geom_step()+
  geom_step(aes(time_days,survival_prob+margin_of_error, color='red'))+
  geom_step(aes(time_days, survival_prob-margin_of_error,color='red'))+
  xlab("Time in Days")+
  ylab("Survival Probability")+
  ggtitle("Survival Probability vs Time in Days With Confidence Interval")+
  theme(legend.position = "none")
```

![](Stat_Learning_Final_Project--2-_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

<br /> Now we can graph the Survival Probability with the confidence
intervals, which are shown in red.

$\textbf{Check your understanding:}$ How would the confidence intervals
change if the sample size (number of patients) was smaller? If the
sample size was larger?

### $\textbf{Cumulative Incidence Curves}$

Cumulative incidence curves can be thought of as the opposite of
survival probability curves. They show the cumulative probability of
experiencing the event of interest, which in this example is death. The
formula for calculating cumulative incidence is 1-S$_t$, or one minus
the survival probability. We can visualize cumulative probability with
the graph below. Notice how it starts around zero and increases over
time, since the cumulative incidence of patient death increases over
time.

``` r
ggplot(kaplan_meier,aes(time_days,1-survival_prob))+
  geom_step()+
  xlab("Time in Days")+
  ylab("Cumulative Incidence")+
  ggtitle("Cumulative Incidence vs Time in Days for Lung Cancer Patients")
```

![](Stat_Learning_Final_Project--2-_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->

<br />

### $\textbf{Log-Rank Test}$

The next method we will use to analyze survival in relation to lung
cancer is the Log-Rank test. This test allows us to evaluate if there is
a significant difference in two or more groups when it comes to
survival. The test will reveal discrepancies in the probability of
survival. The steps for this hypothesis test are broken down below.

1.  **Set Up Hypotheses**

$H_o:$ Survival time is identical between senior citizens and non-senior
citizens.

$H_a:$ Survival time is not identical between senior citizens and
non-senior citizens.

2.  **Calculate Test Statistic**

We will use $\chi^2$ as our test statistic. We utilize the calculation
of the $\chi^2$ statistic to be able to incorporate the *extent of
exposure* and *total number of groups* for each group. For this example,
we will only be summing over two groups, but this method can be expanded
to include multiple groups. The $\chi^2$ statistic can be calculated as
follows:

$$
\chi^2 = \frac{(|O_1-E_1|-\frac{1}{2})^2}{E_1}+\frac{(|O_2-E_2|-\frac{1}{2})^2}{E_2}
$$

where $O_1$ and $O_2$ are total number of deaths for each group, and
$E_1$ and $E_2$ are the extent of exposure measures for each group. The
extent of exposure is representative of the expected number of deaths
among each group.

3.  **Make a Conclusion Off Test Statistic**

The $\chi^2$ calculated will be compared to a $\chi^2$ distribution on 1
degree of freedom. As with most statistical tests, we will make a
decision comparing the p-value to $\alpha=0.05$.

For this example, we will test if there is a difference in survival
probability between senior citizens and everyone else. We will define a
senior citizen as someone who is 62 or older.

``` r
# create new senior citizen variable
cancer_data$senior_cit <- with(cancer_data, ifelse(age < 62, 0, 1))
```

To implement this code, we need the `survdiff()` function. Just like
every other function we have utilized thus far, we will need to input a
formula in the function. As a reminder, when `senior_cit=0`, the patient
is under 62 years of age. When `senior_cit=1`, the patient is at least
62 years of age.

``` r
# log-rank test
diff_age <- survdiff(Surv(time, status) ~ senior_cit, data=cancer_data)
diff_age
```

    ## Call:
    ## survdiff(formula = Surv(time, status) ~ senior_cit, data = cancer_data)
    ## 
    ##                N Observed Expected (O-E)^2/E (O-E)^2/V
    ## senior_cit=0  99       68     74.2     0.512     0.934
    ## senior_cit=1 129       97     90.8     0.418     0.934
    ## 
    ##  Chisq= 0.9  on 1 degrees of freedom, p= 0.3

The output above indicates that there is no significant difference
between survival time of senior citizens and non-senior citizens. We
fail to reject our null hypothesis with a $\chi^2 = 0.9$ and p-value
$=0.3$. We can explore another group comparison and its effect on
survival time.

Allow us to also run a Log-Rank test to determine if there is a
significant difference in survival time for those who have lost versus
gained weight in the last 6 months.

``` r
# examine spread of wt.loss
median_wtloss <- median(cancer_data$wt.loss, na.rm=TRUE)

# create new weight loss variable
cancer_data$wt.cutoff <- with(cancer_data, ifelse(wt.loss < 0, 0, 1))
```

Again, we need the `survdiff()` function. As a reminder, when
`wt.cutoff=0`, the patient has gained weight in the last 6 months
whereas when `wt.cutoff=1`, the patient has lost weight in the past 6
months.

``` r
# log-rank test
diff_weight <- survdiff(Surv(time, status) ~ wt.cutoff, data=cancer_data)
diff_weight
```

    ## Call:
    ## survdiff(formula = Surv(time, status) ~ wt.cutoff, data = cancer_data)
    ## 
    ## n=214, 14 observations deleted due to missingness.
    ## 
    ##               N Observed Expected (O-E)^2/E (O-E)^2/V
    ## wt.cutoff=0  27       20     19.7  0.005393   0.00624
    ## wt.cutoff=1 187      132    132.3  0.000802   0.00624
    ## 
    ##  Chisq= 0  on 1 degrees of freedom, p= 0.9

This test output indicates that there is not a significant difference in
survival times between those who have lost weight in the last 6 months
and those who have gained. With a $\chi^2=2.9$ and a p-value $=0.09$, we
fail to reject the null at $\alpha=0.05$. </br>

**Test Your Understanding:** What values go into the calculation of the
$\chi^2$ statistic when performing a Log-Rank test?

### $\textbf{Cox Regression}$

Let’s start with a breif review of probability distributions. For the
survival analysis, let’s consider the Poisson distribution, which
represents the number of times an event occurs given some period of
time. On the other hand, the exponential distribution is about how much
time is needed approximately until an event occurs. Recall the
Cumulative Distribution function of the exponential distribution:</br>

<center>
$f(t)=P(T\leq{t})=1-e^{-\lambda t}$
</center>

</br>

As discussed above, the ideology behind the survival analysis is
understanding the survival rate past a certain time point, which, in
mathematical terms would be $P(T>t)$. Hence, for the survival function
we get: </br>

<center>
$S(t) = P(T>t)=1-P(t\leq t) = 1-f(t) = 1 - [1-e^{-\lambda t}] = e^{-\lambda t}$
</center>

</br>

<center>
$S(t) = e^{-Hazard \text{ } \cdot \text{ } t}$
</center>
<div>

The multiple regression model of Cox or Hazards Survival Regression
analysis is used for censored data in survival analysis. The model
assumes the hazards of any two patients at a given time are
proportional. In other words, despite the real life possibility of the
hazard having a different pattern among different patients, the model
assumes that the changes in the hazard of one patient will be
proportional to the changes in the hazard of another patient. </br> The
Cox Regression Model has the advantage of assessing how influential each
of the predictors is in the model by generalizing a pattern of
covariance within the hazard. </br> With $t$ being defined as the hazard
at time $t$ after a starting point of $0$, for an individual with
variables $z$ = $(z_1, z_2…, z_p)$ which are also recorded from time
$0$, the model has the following form:</br>

<center>
$\lambda(t,z) = \lambda_0(t) e^{b_1z_1 + ... + b_iz_i + ... + b_pz_p}$
</center>

</br> Similar to the regular regression models, the regression
coefficients $b_i$ represent the amount by which the predictor variable
$z_i$ contributes to the hazard. The contribution is small or large
correspondingly depending on the numeric value of $b_iz_i$. </br>

It is possible to estimate and conduct significance testing of a given
Cox model with the help of the concept of likelihood. It provides the
probability of the observed data being explained by a certain model.
</br>

**Test Your Understanding:** What is a disadvantage of Cox Proportional
Hazards Model?

# $\textbf{Calculating Survival Analysis Using R Packages}$

### $\textbf{Kaplan-Meier Method}$

This method utilizes a survival function formatted to use time as an
input to return a probability of survival at any time. This
non-parametric method is rooted in a few basic assumptions. As
previously mentioned, the method assumes that survival probability is
the same for those from different entry times and for those with a
different censoring status. Calculating the product of survival for all
previous times will result in the probability of survival for the
current time. The survival function is as follows:

$$
S(t) = \prod_{i=0}^{t-1}(1-\frac{d_i}{n_i})
$$

where $t$ is the current time being evaluated, $d_i$ is the number of
deaths that occurred at time $i$, and $n_i$ is the number of individuals
that have survived up to time $i$. One might introduce the name Thomas
Bayes to reference the fact that the equation above utilizes *prior*
information when calculating the output at each step of time.

Within the `survival` package we will use the `survfit()` function to
fit a survival curve. This function requires a formula as an input
parameter. The formula should use a categorical variable in the data
with a small number of categories. For this reason, we will use `sex` to
create a survival curve in terms of `time` and `status`. As we will see,
the plot is formatted so that the x-axis represents time, the y-axis
represents probability of survival, and curves are grouped in terms of
the variable included in the fit formula.

``` r
# fit survival curve
survival_fit <- survfit(Surv(time, status) ~ sex, data=cancer_data)
# print a summary of observations both sex groups
print(survival_fit)
```

    ## Call: survfit(formula = Surv(time, status) ~ sex, data = cancer_data)
    ## 
    ##         n events median 0.95LCL 0.95UCL
    ## sex=1 138    112    270     212     310
    ## sex=2  90     53    426     348     550

Once we have created the `survival_fit` object, we have the ability to
access various components. Below, we have printed a data frame that
contains the lower and upper bounds of the confidence interval in which
the survival probability is contained at each time mark.

``` r
# print lower and upper bounds in a tibble
low_up <- tibble(time = survival_fit$time,
       lower = survival_fit$lower,
       upper = survival_fit$upper)
low_up                                       # print results
```

    ## # A tibble: 206 × 3
    ##     time lower upper
    ##    <dbl> <dbl> <dbl>
    ##  1    11 0.954 1    
    ##  2    12 0.943 0.999
    ##  3    13 0.923 0.991
    ##  4    15 0.913 0.987
    ##  5    26 0.904 0.982
    ##  6    30 0.894 0.977
    ##  7    31 0.885 0.972
    ##  8    53 0.867 0.961
    ##  9    54 0.858 0.956
    ## 10    59 0.850 0.950
    ## # … with 196 more rows

#### Survival Curve

Within the `survminer` package we can use the `ggsurvplot()` function to
actually visualize this fit on a 2-D plot. There are a few input
parameters that go into constructing this plot. Reference the
documentation and our comments to understand the options surrounding
parameter assignment.

``` r
# plot curve
ggsurvplot(survival_fit, pval=TRUE,        #display p-value
           conf.int=TRUE,                  #display confidence intervals
           linetype=1,                     #line type by ph.ecog
           pallete=c('#E57200','#232D4B'), #manually select color pallete
           risk.table=TRUE)                #print risk table below by group
```

![](Stat_Learning_Final_Project--2-_files/figure-gfm/unnamed-chunk-18-1.png)<!-- -->

The plot above displays the probability of survival in terms of time. As
a reminder, `sex=1` represents males while `sex=2` represents females.
Overall, the female group tends to have a slightly higher probability of
survival regardless of time. Across both groups, the probability appears
to fall to nearly 0 after 1000 days.

Risk is another measure that we can evaluate that goes hand in hand with
probability of survival. A risk table is displayed below the plot that
indicates specific counts of the number of survivors that are at risk
for each group at the given point in time.

#### Cumulative Hazard Curve

Similar to the previous mention in this tutorial of Kaplan-Meier, we can
model a similar curve but with cumulative hazard against time. A good
way to conceive of cumulative hazard is to imagine the impact of lung
cancer on death in terms of time. We can say that the y-axis is
representative of the cumulative risk of a person dying from lung cancer
at a specific time. This is why we notice an increasing curve in the
plot below.

The cumulative hazard function is simply the negative log of the
survival function.

$$
H(t) = -log(S(t))
$$

``` r
# plot curve
ggsurvplot(survival_fit,
           conf.int=TRUE,                  #display confidence intervals
           linetype=1,                     #line type by ph.ecog
           pallete=c('#E57200','#232D4B'), #manually select color pallete
           risk.table=TRUE,                #print risk table below by group
           fun='cumhaz')                   #cumulative hazard
```

![](Stat_Learning_Final_Project--2-_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

### $\textbf{Cox-Proportional Hazard Model}$

Building off of the cumulative hazard curve that we just plotted, we
will now be exploring the Cox-Proportional Hazard model to more closely
evaluate specific predictors’ impact on survival. So far, the only
variable that we have isolated is `sex`, and that was only to visually
separate survival curves into groups.

Unlike the Kaplan-Meier method, the Cox-Proportional Hazard model is a
semi-parametric method meaning it contains finite-dimensional and
infinite-dimensional parameters. This detail isn’t crucial to
understanding the general operation, but should still be noted.

#### Coxph

Since we have only looked at the impact of `sex` alone on survival, we
will now consider multiple variables. The `coxph()` function is
available through the `survival` package and will again require a
formula input as shown below.

``` r
# fit cox model
cox_fit <- coxph(Surv(time, status) ~ age+sex+ph.ecog+meal.cal+wt.loss, data=cancer_data)
cox_fit
```

    ## Call:
    ## coxph(formula = Surv(time, status) ~ age + sex + ph.ecog + meal.cal + 
    ##     wt.loss, data = cancer_data)
    ## 
    ##                coef  exp(coef)   se(coef)      z        p
    ## age       8.693e-03  1.009e+00  1.124e-02  0.773 0.439303
    ## sex      -5.400e-01  5.827e-01  1.988e-01 -2.716 0.006602
    ## ph.ecog   5.152e-01  1.674e+00  1.483e-01  3.474 0.000513
    ## meal.cal -4.499e-05  1.000e+00  2.512e-04 -0.179 0.857865
    ## wt.loss  -1.117e-02  9.889e-01  7.548e-03 -1.480 0.138841
    ## 
    ## Likelihood ratio test=22.34  on 5 df, p=0.0004517
    ## n= 170, number of events= 123 
    ##    (58 observations deleted due to missingness)

In the model above, the variables `age`, `sex`, `ph.ecog`, `meal.cal`,
and `wt.loss` are all included. The most important piece of input to
extract from this output is the $exp(\text{coef})$ term that exists for
each variable. This is the Hazard Ratio (HR) and can be interpreted as
follows. To remind ourselves, hazard relates to the chances a patient
has of dying.

-   HR = 1: Variable has no effect on hazard

-   HR \< 1: Variable influences a reduction in hazard

-   HR > 1: Variables influences an increase in hazard

We have the ability to evaluate any variable in relation to hazard here.
The interpretation for each variable is:

-   `age`: Nearly 1 indicates no effect on hazard.

-   `sex`: Less than 1 indicates a reduction in hazard. Females tend to
    display a lower risk in dying due to lung cancer.

-   `ph.ecog`: Greater than 1 indicates an increase in hazard. The more
    stationary or closer to bedridden a patient is, the most at risk
    they are of dying due to lung cancer.

-   `meal.cal`: Nearly 1 indicates no effect on hazard.

-   `wt.loss`: Less than 1 indicates a reduction in hazard. The more
    weight a patient has lost, the less at risk they are of dying due to
    lung cancer.

Overall, these results are not terribly surprising. It is interesting
that not every variable had an impact. Both `age` and `meal.cal`
indicated no large risk to overall survival probability in relation to
lung cancer. <br />

**Test Your Understanding:** How can we interpret hazard in relation to
specific variables? <br /> <br />

### $\textbf{Further Extensions}$ <br />

For a small percentage of cases (for example: a clinical trial where
participants drop out) it is acceptable to apply some censoring. It is
common to encounter scenarios where not all the events of interest occur
in the duration of the study, leading to some gaps in the data (it is
important to mention that in survival analysis ‘censored data’ is
different from ‘missing data’). The censored participants still
contribute equally with the last interval of their availability.
However, the survival model will not work very well unless enough
non-censored observations are provided.

As much as the terminology of survival analysis implies the negative
nature of the target--survival, hazard, failure, etc--in many
applications the outcome of interest is a good thing and we are
predicting how soon it will occur. We can use customer analytics to
understand customer behavior in as marketing, or sales . Survival
analysis can be used in human resources to understand the pattern of
filling in open positions or defining the time it takes for promotions.
Another application can be product analytics, to understand the
potential amount of time for customers to upgrade to the latest version
of some product, say, a laptop, or some other technology.

### $\textbf{Conclusion}$ <br />

We’ve explored the primary applications and implementations of survival
analysis, in short:

-   The Kaplan-Meier method: the primary statistic used to calculate
    survival probability for a specified point in time.

-   The Log-Rank Test: statistical test for comparing survival time
    between groups – allows us to assess whether significant differences
    exist between populations.

-   Cox Regression: model for determining how different variables impact
    prediction for survival time

Taken together, survival analysis methods provide a powerful mechanism
for both anticipating and understanding the time until an entity or
group of entities experiences an event. This is of particular value in
cases where the factors influencing the outcome of interest are numerous
and complex, and is widely used in the fields of medicine, economics and
sociology. <br />

### $\textbf{Resources}$ <br />

The following references were used to construct this tutorial, and have
additional information that may be helpful to viewers. <br />

Survival analysis \| The BMJ. (n.d.). The BMJ \| The BMJ: Leading
General Medical Journal. Research. Education. Comment. Retrieved
December 13, 2022, from
,<https://www.bmj.com/about-bmj/resources-readers/publications/statistics-square-one/12-survival-analysis>
<br /> <br /> Babucea, A.-G., & Danacica, D.-E. (n.d.). USING SURVIVAL
ANALYSIS IN ECONOMICS. Comparing Survival Curves. (n.d.). Retrieved
December 13, 2022, from
<https://sphweb.bumc.bu.edu/otlt/mph-modules/bs/bs704_survival/BS704_Survival5.html#:%5C~:text=The%20log%20rank%20test%20is,identical%20(overlapping)%20or%20not>
<br /> <br /> Goel, M. K., Khanna, P., & Kishore, J. (2010).
Understanding survival analysis: Kaplan-Meier estimate. International
Journal of Ayurveda Research, 1(4), 274–278.
<https://doi.org/10.4103/0974-7788.76794> <br /> <br /> Log Rank Test—An
overview \| ScienceDirect Topics. (n.d.). Retrieved December 13, 2022,
from
<https://www.sciencedirect.com/topics/medicine-and-dentistry/log-rank-test>
<br /> <br /> Nonparametric and Semiparametric Modeling—U-M School of
Public Health. (n.d.). Retrieved December 13, 2022, from
<https://sph.umich.edu/biostat/faculty-research/nonparametric-semiparametric-modeling.html#:~:text=A%20semiparametric%20model%20is%20intermediate,over%20the%20past%20two%20decades>
<br /> <br /> Regularized Cox Regression. (n.d.). Retrieved December 13,
2022, from <https://glmnet.stanford.edu/articles/Coxnet.html> <br />
<br /> Stel, V. S., Dekker, F. W., Tripepi, G., Zoccali, C., & Jager, K.
J. (2011). Survival Analysis II: Cox Regression. Nephron Clinical
Practice, 119(3), c255–c260. <https://doi.org/10.1159/000328916> <br />
<br /> Survival Analysis. (n.d.). Retrieved December 13, 2022, from
<https://sphweb.bumc.bu.edu/otlt/mph-modules/bs/bs704_survival/BS704_Survival_print.html>
<br /> <br /> Survival Analysis: An Example. (n.d.). Retrieved December
13, 2022, from
<https://quantdev.ssri.psu.edu/sites/qdev/files/12_SurvivalAnalysis_2020_0408.html>
<br /> <br /> Survival Analysis Basics—Easy Guides—Wiki—STHDA. (n.d.).
Retrieved December 13, 2022, from
<http://www.sthda.com/english/wiki/survival-analysis-basics#:~:text=In%20cancer%20studies%2C%20most%20of,effect%20of%20variables%20on%20survival>
<br /> <br /> Survival Analysis of Lung Cancer Patients \| Kaggle.
(n.d.). Retrieved December 13, 2022, from
<https://www.kaggle.com/code/saychakra/survival-analysis-of-lung-cancer-patients/notebook#Application-of-the-Cox-proportional-Hazard-model>
