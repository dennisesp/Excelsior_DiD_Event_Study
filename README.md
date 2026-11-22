# Replicating the Excelsior Scholarship's Enrollment Effects

*A partial replication of Joseph E. Howard's (2026) dissertation, **Excelsior Scholarship and College Enrollment in New York: Evidence of Unexpected Gains**.*

## Project Overview

New York introduced the Excelsior Scholarship in 2017 to help eligible residents cover tuition at public colleges and universities. In his dissertation, Howard (2026) examines whether enrollment changed following the program's introduction, using Pennsylvania as a comparison state. Through his analysis Howard found that enrollment in New York went up for Pell grant eligible students following the excelsior scholarship program's introduction. 

For this project, I recreate Howard's main findings for public institutions using the Integrated Postsecondary Education Data System (IPEDS) and Bureau of Labor Statistics (BLS) data from 2012 through 2019. Just like the original study did, I analyzed data  on first-time, full-time undergraduate enrollment and enrollment among Pell Grant recipients across Pennsylvania and New York public institutions. 

My main data limitations compared to Howard's original study was that I could not obtain Penn State's disaggregated campus-level. This meant that the instiution count for the control state, Pennsylvania, was lower than the count in Howard's study. Still, I was able to replicate Howard's methods, and nearly match his outcomes. 

## Data

I pulled the data and completed the cleaning process in my data cleaning notebook, found in the folder labeled "Data Cleaning" . I organized the raw files into three groups:

1. **IPEDS data:** Enrollment, Pell Grant recipients, tuition and fees, and institutional characteristics from the Integrated Postsecondary Education Data System.
2. **Bureau of Labor Statistics (BLS) data:** County-level unemployment rates.
3. **County FIPS data:** A county-name and FIPS-code file shared by an OpenDataPhilly user. I used this to connect the county names in my IPEDS files to the FIPS codes needed for the BLS merge. The IPEDS extracts I downloaded provided county names without the codes I needed.

| Source | Data I used | Years |
| --- | --- | --- |
| [IPEDS, National Center for Education Statistics](https://nces.ed.gov/ipeds/use-the-data) | Institution identifiers, enrollment counts, Pell Grant counts, tuition and fees, institutional control, and location | Fall 2012–2019 |
| [BLS, Local Area Unemployment Statistics](https://www.bls.gov/lau/) | County unemployment rates | 2011–2018, lagged one year |
| OpenDataPhilly user-contributed county FIPS file | County names and FIPS codes used to match IPEDS and BLS | County crosswalk |

For the analysis, my unit of observation is an **institution-year**. Where New York public institutions and Pennsylvania public institutions as the treatment and control groups, respectively. The treatment is the introduction of the Excelsior scholarship in 2017.

My final sample has **111 public institutions**, including **74 in New York and 37 in Pennsylvania**, observed over eight years (Different than Howard because he had 125 institutions total, and 51 in Pennsylvania).

### Main Variables

| Variable | Definition |
| --- | --- |
| `unitid` | Identify institutions, add institution fixed effects, and cluster standard errors |
| `fall_year` | Identify the year and add year fixed effects |
| `state` | Distinguish New York from Pennsylvania |
| `Grand total` | First-time, full-time undergraduate student enrollment count|
| `Number of full-time first-time undergraduates awarded Pell grants` | First-time, full-time Pell Grant awarded students  count |
| `Published in-state tuition and fees` | Represents the logged tuition variable for adjusted models |
| `unemp_lag` | Control for county unemployment in the preceding year |

These are the main variables I use in the analysis notebook.

## Data Cleaning

My cleaning process is in full detail in the Data cleaning Python notebook tittled "Data_cleaning.ipynb. Where I started with separate annual IPEDS files and BLS data and merged them into one institution-year panel.

1. **Merging IPEDS files.** I merged the annual Pell Grant, race and gender enrollment, and tuition files using `unitid`. I also standardized the column names across years.
2. **Creating the panel.** I combined the annual datasets into one file and added `fall_year`.
3. **Adding county FIPS codes.** I matched the IPEDS county names to the county FIPS crosswalk sourced from an OpenDataPhilly user.
4. **Merging unemployment data.** I cleaned the BLS files and matched unemployment rates to institutions using county FIPS codes. I lagged unemployment by one year (`unemp_year = fall_year - 1`).
5. **Keeping public institutions.** I used the IPEDS institutional-control variable to identify public colleges and universities.
6. **Applying exclusions.** I excluded specialty institutions identified in Howard's dissertation and removed Penn State campuses because of the 2019 data limitation.
7. **Saving the cleaned dataset.** I exported the final panel as `panel_clean.csv` for the analysis notebook.

### Why I Excluded Penn State

Beginning in 2019, Penn State changed how it reported campus enrollment data in IPEDS, reporting all data from separate campuses under separate campus names before 2018, and then began merging them into one "Penn State" name in 2019. Howard obtained supplemental campus-level data directly from Penn State to keep 2019 analysis consistent with the other years, but I did not have access to those records. So I chose to remove all Penn State campuses so I could keep a consistent set of institutions across years without filling in missing values.

This is the exclusion I use for my final sample:

```python
psu = panel["institution name"].str.startswith(
    "Pennsylvania State University",
    na=False
)

panel = panel.loc[~psu].copy()
```

The exclusion changes the Pennsylvania comparison group, so my results will not match Howard's estimates exactly.

## Analysis

My analysis can be found in [`Excelsior_Replication_Step_by_Step.ipynb`](Excelsior_Replication_Step_by_Step.ipynb). In that Python notebook, I  perform Howards' DiD and Event study as well as replicated some of his other figures, and compare my results to his tables 10-13. 




### 1. Checking Parallel Trends

I first plotted average enrollment in New York and Pennsylvania between 2012 and 2019. 

**First-Time, Full-Time Undergraduate Enrollment**

![FTFT Enrollment Trends](Figures/ftft_enrollment_trends.png)

**Pell Grant Recipient Enrollment**

![Pell Grant Enrollment Trends](Figures/pell_enrollment_trends.png)




I then tested whether the two states had significantly different linear enrollment trends before treatment. For the statistical test, I interact the New York indicator with a linear time trend and include institution and year fixed effects. I run the test with treatment beginning in both 2017 and 2018.

<td valign="top">
  <a href="Figures/Parallel_Trends_Stats.png">
    <img
      src="Figures/Parallel_Trends_Stats.png"
      alt="My parallel trends statistical results"
      width="100%"
    >
  </a>
</td>

None of my pretreatment slope tests were statistically significant at the 5% level. I therefore fail to reject the null hypothesis that the pretreatment slopes are equal, which provides some support for the parallel trends assumption.





### 2. Difference-in-Differences

Next, I estimate two-way fixed-effects DiD models:

$$
Y_{it} = \alpha + \gamma_i + \lambda_t + \beta(\text{NY}_i \times \text{Post}_t) + \varepsilon_{it}
$$

Here, $Y_{it}$ is enrollment at institution $i$ in year $t$, while $\gamma_i$ and $\lambda_t$ represent institution and year fixed effects. The coefficient $\beta$ captures the estimated change in New York enrollment relative to Pennsylvania after treatment.

Following Howard's approach, I run separate models with treatment starting in **2017** and **2018**. He analyzed both years because he wasn't sure if the 2017 cohort would've been impacted by the scholarship so soon after its announcement.



### 3. Event Study

Lastly, I estimate enrollment differences separately for each year, using **2016 as the reference year** because it was the last year before the Excelsior Scholarship was introduced in 2017 allowing  me to examine how enrollment differences between New York and Pennsylvania changed before and after the program's introduction.

```math
Y_{it} = \alpha + \gamma_i + \lambda_t + \sum_{k \ne 2016} \delta_k (\text{NY}_i \times \mathbf{1}[t=k]) + \varepsilon_{it}
```

I use the estimates from 2012–2015 to examine pretreatment differences and the estimates from 2017–2019 to examine changes following Excelsior's introduction. I also plot the coefficients with 95% confidence intervals.

The event-study models reported below include institution and year fixed effects and institution-clustered standard errors (they don't include the additional tuition and unemployment covariates.)

## Main Findings

### Difference-in-Differences Results

After adding tuition and lagged county unemployment, my estimated treatment coefficients decreased across all four models. The coefficients remained positive, although none were statistically significant at the 5% level.

Overall, I find a similar pattern to Howard's DiD results demonstrated by standard errors close to or larger than the coefficients and 95% confidence intervals that include zero for every adjusted model (p = 0.196 to 0.444).

#### DiD Tables: My Findings and Howard's Tables 10 and 11

**Treatment begins in 2017**

<table>
<tr><th width="50%">My 2017 DiD results</th><th width="50%">Howard's Table 10</th></tr>
<tr>
<td valign="top">
<table>
<tr><th>Outcome</th><th>Without covariates</th><th>With covariates</th></tr>
<tr><td>FTFT enrollment</td><td align="right">82.99</td><td align="right">38.26</td></tr>
<tr><td>Pell FTFT enrollment</td><td align="right">50.64</td><td align="right">27.14</td></tr>
</table>
<p>I find positive estimates for both outcomes, but neither adjusted estimate is significant at the 5% level.</p>
</td>
<td valign="top"><a href="Figures/howard_table10.png"><img src="Figures/howard_table10.png" alt="Howard's 2017 DiD results in Table 10" width="100%"></a></td>
</tr>
</table>

**Treatment begins in 2018**

<table>
<tr><th width="50%">My 2018 DiD results</th><th width="50%">Howard's Table 11</th></tr>
<tr>
<td valign="top">
<table>
<tr><th>Outcome</th><th>Without covariates</th><th>With covariates</th></tr>
<tr><td>FTFT enrollment</td><td align="right">85.78</td><td align="right">58.33</td></tr>
<tr><td>Pell FTFT enrollment</td><td align="right">46.11</td><td align="right">30.30</td></tr>
</table>
<p>The estimates remain positive when I move treatment to 2018, but neither adjusted coefficient is significant at the 5% level.</p>
</td>
<td valign="top"><a href="Figures/howard_table11.png"><img src="Figures/howard_table11.png" alt="Howard's 2018 DiD results in Table 11" width="100%"></a></td>
</tr>
</table>



### Event-Study Results

My event-study estimates show positive enrollment changes in New York following Excelsior's introduction.

For first-time, full-time undergraduate enrollment, all three post-treatment coefficients were statistically significant at the 5% level. The estimates suggest increases of approximately 64, 100, and 102 students per New York public institution in 2017, 2018, and 2019, respectively, relative to Pennsylvania.

I observed similar results for Pell Grant recipients. The estimated increases were approximately 44 students in 2017, 57 in 2018, and 49 in 2019. The 2017 and 2018 coefficients were significant at the 5% level, while the 2019 estimate was marginally significant at the 10% level.

None of the individual pretreatment event-study coefficients were statistically significant at the 5% level.

My results are generally consistent with Howard's main event-study findings. The differences in coefficient magnitudes may be related to my exclusion of Penn State and other differences in how the data were constructed.

#### Event-Study Tables: My Findings and Howard's Tables 12 and 13

**First-time, full-time enrollment**

<table>
<tr><th width="50%">My FTFT event-study results</th><th width="50%">Howard's Table 12</th></tr>
<tr>
<td valign="top">
<table>
<tr><th>Year</th><th>Estimate</th><th>p-value</th></tr>
<tr><td>2017</td><td align="right">63.65</td><td align="right">0.004</td></tr>
<tr><td>2018</td><td align="right">100.01</td><td align="right">0.002</td></tr>
<tr><td>2019</td><td align="right">102.05</td><td align="right">0.017</td></tr>
</table>
<p>My post-treatment estimates are positive and significant at the 5% level in all three years, similar to Howard's findings.</p>
</td>
<td valign="top"><a href="Figures/howard_table12.png"><img src="Figures/howard_table12.png" alt="Howard's FTFT event study table" width="100%"></a></td>
</tr>
</table>

**Pell Grant enrollment**

<table>
<tr><th width="50%">My Pell FTFT event-study results</th><th width="50%">Howard's Table 13</th></tr>
<tr>
<td valign="top">
<table>
<tr><th>Year</th><th>Estimate</th><th>p-value</th></tr>
<tr><td>2017</td><td align="right">43.97</td><td align="right">0.001</td></tr>
<tr><td>2018</td><td align="right">56.69</td><td align="right">0.005</td></tr>
<tr><td>2019</td><td align="right">48.84</td><td align="right">0.058</td></tr>
</table>
<p>I find statistically significant increases in 2017 and 2018. The 2019 estimate is marginally significant at the 10% level.</p>
</td>
<td valign="top"><a href="Figures/howard_table13.png"><img src="Figures/howard_table13.png" alt="Howard's Pell FTFT event study table" width="100%"></a></td>
</tr>
</table>

#### Event-Study Figures: My Estimates and Howard's Figures

**First-time, full-time enrollment**

<table>
<tr><th width="50%">My FTFT event-study figure</th><th width="50%">Howard's FTFT event-study figure</th></tr>
<tr>
<td valign="top"><a href="Figures/event_study_ftft.png"><img src="Figures/event_study_ftft.png" alt="My FTFT event study with 95 percent confidence intervals" width="100%"></a></td>
<td valign="top"><a href="Figures/howard_event_study_ftft.png"><img src="Figures/howard_event_study_ftft.png" alt="Howard's FTFT event study figure" width="100%"></a></td>
</tr>
</table>

**Pell Grant enrollment**

<table>
<tr><th width="50%">My Pell FTFT event-study figure</th><th width="50%">Howard's Pell FTFT event-study figure</th></tr>
<tr>
<td valign="top"><a href="Figures/event_study_pell.png"><img src="Figures/event_study_pell.png" alt="My Pell event study with 95 percent confidence intervals" width="100%"></a></td>
<td valign="top"><a href="Figures/howard_event_study_pell.png"><img src="Figures/howard_event_study_pell.png" alt="Howard's Pell FTFT event study figure" width="100%"></a></td>
</tr>
</table>

#### Reading the Figures

I compare my figures with Howard's figures for the same outcomes and treatment dates. I also keep the regression tables together above so I can compare the coefficients directly. Differences in our institutional samples are important when interpreting any differences in the estimates.


## Limitations

My analysis has several limitations that are important for interpreting the findings.

- **Penn State:** I excluded all Penn State campuses because I did not have the disaggregated 2019 data Howard used. This changes the Pennsylvania sample and limits how closely I can reproduce his estimates.
- **Student migration:** I did not recreate Howard's in-state enrollment models because I did not have the migration data needed for those specifications.
- **Causal interpretation:** My analysis depends on the parallel trends assumption and the absence of other shocks that affected the two states differently. The pretreatment tests provide some support, but they cannot establish those assumptions.
- **Scope:** I focused on the main public-institution findings. I did not recreate every outcome, institutional sector, or additional analysis in the dissertation.
