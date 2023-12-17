---
layout:     post
title:      "University Traffic Fines: Trends in Weather & Air Quality - Data Collection"
description: "An exploratory data science project to research how changes in weather and air quality affect the incentives of both students and university parking police"
date:    2023-11-13
author:     Sam Lee
image: "/img/byu-traffic-citations/provo.JPG"
published: true 
tags:
- python 
- EDA
- Data Science
URL: "/2023/11/13/byu-traffic-citations/"    
---

All code and progress for this project is kept up to date at [https://github.com/SamLeeBYU/BYUTrafficCitations](https://github.com/SamLeeBYU/BYUTrafficCitations).

## Introduction

The idea for this project sparked when I received yet another parking citation from BYU during the spring semester of 2023. It wasn't until I went to pay the citation where I realized BYU's citation database [1] was scrapable. Though BYU hides certain citation-specific details for privacy purposes, the general information of every citation scraped is available for anyone with BYU credentials: Notable information such as the time of a citation, plate/vin number, and fine amount could all be parsed by iterating through a combination of citation numbers. Through a process which I explain thoroughly [here](https://samleebyu.github.io/2023/09/29/selenium-best-practices/), I scraped the traffic citations going back to July 2012 to present (October 2023 at the time of scraping).

## Problem and Motivation

Upon looking at the aggregate daily trends in the number of daily fines over time, I realized that there seemed to be explainable trends in the data--it was more than just a random walk. Surely the number of students who are obligated to go to campus on any given day of the week influenced the large fluctuations in number of tickets given out--things like enrollment, whether the day was a holiday or if was a day during final exams, what day of the week it was, or just the general trend in enrollment over time--could explain the long-run fluctuations. However, I wanted to know if the short-run fluctuations could be explained. I wanted to know if changes from day to day or week to week depended on the incentives of students' transportation choices in the midst of weather and air quality effects. On the contrary, were the small fluctuations I was seeing from day to day and week to week just simply due to random error? 

## Data Collection

To investigate these questions, I needed to merge the traffic citations data I scraped to past enrollment data and other data that controls for changes in long-run shifts. In addition, I needed to curate historical weather data and historical air quality data for each day and merge that with the traffic citations data as well.

#### Enrollment Data and Calendar Data

In order to control for obvious large student demand changes, I needed to include the changes in BYU student enrollment over time as well as the indicators to denote whether students are on holiday breaks or if its exam season. No public data base/website has this information. I went to several sources within BYU's HBLL, and they eventually directed me to their head archivist, Cory Nimer, who directed me to some historical calendar records [6]. I also obtained past enrollment figures going back to 2014 from [BYU Research & Reporting Enrollment Services](https://tableau.byu.edu/#/site/BYUCommunity/views/UniversityEnrollmentStatistics/EnrollmentStatistics) [5]. Modern data scraping methods were not applicable here. I manually curated it into a csv file called [enrollment.csv](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/enrollment.csv).

#### Collecting Weather Data for Provo, UT

To obtain weather data for Provo, UT I used the climate API, [Open-Meteo](https://open-meteo.com/en/docs/climate-api). Open-Mateo's climate API is extremely easy to use. For requests of smaller-sizes, Open-Meteo provides free API keys.

Here's a real example of how I used it in my code:

```
import pandas as pd
import requests

base_url = "https://archive-api.open-meteo.com/v1"

citations = pd.read_csv("ParkingCitationsEncrypted.csv")
citations["IssuedDate"] = pd.to_datetime(citations["IssuedDate"])

endpoint = f"/archive?latitude=40.2338&longitude=-111.6585&start_date={citations['IssuedDate'].dt.date.values[0].strftime('%Y-%m-%d')}&end_date={citations['IssuedDate'].dt.date.values[len(citations['IssuedDate'])-1].strftime('%Y-%m-%d')}&daily=temperature_2m_max,temperature_2m_min,temperature_2m_mean,rain_sum,snowfall_sum,wind_speed_10m_max&timezone=America%2FDenver"
url = base_url + endpoint

weather_data = pd.DataFrame(requests.get(url).json()["daily"])

weather_data = weather_data.rename(columns={'time': 'IssuedDate'})
weather_data["IssuedDate"] = pd.to_datetime(weather_data["IssuedDate"])
weather_citations = pd.merge(citations, weather_data, how="left")
```

This example uses the endpoint "archive" to fetch historical weather data. I fill in necessary latitude and longitude coordinates for Provo, UT and request the the weather metrics available on Open-Meteo. The date parameters are simply filled in to align with the starting and ending dates of *citations* data set. (See [Weather.ipynb](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/Weather.ipynb)). I save the API response to [weather.json](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/weather.json) for convenience to be referred to later.

#### Collecting Air Quality Data 

Collecting historical air quality for Provo, UT proved to be a bit more challenging. The APIs I found with that provided free services had either limited historical data (they only went back a certain amount of years), or they provided limited requests. Given my limited funds for this research project I looked elsewhere and I decided to make an attempt at extracting the raw data kept by the Utah Department of Environment Quality [3]. The UDEQ has historical records for key air pollutants ($CO$, $NO_2$, $O_3$, $PM 10$, and $PM 2.5$) from the years going back to 2000. I was able to convert the corresponding tables of pollutant data--which contains daily maximum values for each pollutant for every day in their archive, though with some missing values--for Provo into images which then I could convert into a pandas data frame with Python. The raw image data files are contained in a directory called [Provo Air Quality Data](https://github.com/SamLeeBYU/BYUTrafficCitations/tree/main/Provo%20Air%20Quality%20Data). I used the [ExtractTable](https://extracttable.com/) API to turn each image into a pandas data frame.

Example code:

```
from ExtractTable import ExtractTable
import pandas as pd

with open('extraction_api_key.txt', 'r') as file:
    api_key = file.read()
    
et_sess = ExtractTable(api_key)

metrics = ["CO", "NO2", "O3", "PM10", "PM25"]
months = ['JANUARY', 'FEBRUARY', 'MARCH', 'APRIL', 'MAY', 'JUNE', 'JULY', 'AUGUST', 'SEPTEMBER', 'OCTOBER', 'NOVEMBER', 'DECEMBER']

image_paths = [
    f"{data_dir}/{metric}-{year}.png"
    for year in range(2014, 2022 + 1)
    for metric in metrics
]

for path in image_paths:
    et_sess.process_file(path, output_format="df")
    et_sess.save_output(data_dir, output_format="csv")
```

This converts each pollutant-specific image obtained from UDEQ into a separate intermediate csv file. The next thing I did was merge all the data sets together to yield a final air quality data set for Provo over the years 2014-2022.

```
metrics = ["CO", "NO2", "O3", "PM10", "PM25"]

tables = [
    f"{data_dir}/{metric}-{year}_table_1.csv"
    for year in range(2014, 2022 + 1)
    for metric in metrics
]

ProvoAQ = pd.DataFrame()

for table in tables:
    data = pd.read_csv(table, header=0, names=[col.upper() for col in pd.read_csv(table, nrows=1).columns])
    data["DAY"] = list(range(1, data.shape[0] + 1))
    
    pivoted = pd.melt(data, id_vars=['DAY'], var_name='MONTH', value_name='DATA')
    pivoted["YEAR"] = int(table.split("_table")[0][-4:])
    pivoted["METRIC"] = table.split("/")[-1].split("-")[0]
    
    if ProvoAQ.empty:
        ProvoAQ = pivoted.copy()
    else:
        ProvoAQ = pd.concat([ProvoAQ, pivoted.copy()]).reset_index(drop=True)
```

These lines of code pivots each table such that day is a column instead of a row; this aligns all the tables regardless of how many days there are in the here and allows me to combine all the pollutant-specific records.

I finish up cleaning up the data set and save it to [ProvoAQ.csv](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/ProvoAQ.csv). See [PDF Extraction.ipynb](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/ipynb) for the rest of the code.

Additionally, EPA danger classifications were obtained through documentation on the EPA's [website](https://airnowtomed.app.cloud.gov/sites/default/files/2020-05/aqi-technical-assistance-document-sept2018.pdf) [4]. I classified the $CO$, $NO_2$, $O3$, $PM10$, and $PM2.5$ values with a level as either "Good", "Moderate", "Unhealthy", "Unhealthy for Sensitive Groups", "Unhealthy", and "Very Unhealthy". I then created a formula to calculate the AQI as specified on EPA documentation [4] and mapped that to the air quality data set.

---

Once I collected and cleaned the air quality data, I pulled in all the data sets (*citations.csv*, *weather.json*, *ProvoAQ.csv*, and *enrollment.csv*), cleaned and aggregated the data set (by day) into a single data set called [Provo.csv](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/Provo.csv). (See [EDA.ipynb](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/EDA.ipynb) for full data cleaning and aggregation process).

Our final data set is a 3282x41 table, here are the first five rows:

| DATE       | Month | Day       | NA_Correction | MaxTemp | MinTemp | MeanTemp | RainPrecip | SnowPrecip | Wind | CO  | NO2 | O3   | PM10 | PM25 | CO_LEVEL | NO2_LEVEL | O3_LEVEL | PM10_LEVEL | PM25_LEVEL | AQI | AQI_LEVEL | Year  | DailyNumFines | NumPaidFines | TotalFineAmount | AvgPaidFine | Fri | Mon | Sat | Sun | Thurs | Tues | Wed | Term   | Enrollment | FullTime | Holiday | Exam |
|------------|-------|-----------|---------------|---------|---------|----------|------------|------------|------|-----|-----|------|------|------|----------|-----------|----------|------------|------------|-----|-----------|-------|---------------|--------------|------------------|-------------|-----|-----|-----|-----|-------|------|-----|--------|------------|----------|---------|------|
| 2014-01-06 | 1     | Monday    | False         | -3.3    | -17.6   | -10.5    | 0.0        | 0.0        | 6.8  | 1.8 | 47.0| 0.022| 47.0 | 13.7 | Good     | Good      | Good     | Good       | Moderate   | 54  | Moderate  | 2014  | 23            | 17           | 386.0            | 22.71       | 0   | 1   | 0   | 0   | 0     | 0    | 0   | Winter | 29642      | 25191    | 0       | 0    |
| 2014-01-07 | 1     | Tuesday   | False         | 1.1     | -11.6   | -4.7     | 0.0        | 0.21       | 5.9  | 1.9 | 47.0| 0.007| 66.0 | 21.0 | Good     | Good      | Good     | Moderate   | Moderate   | 70  | Moderate  | 2014  | 52            | 46           | 1238.0           | 26.91       | 0   | 0   | 0   | 0   | 0     | 1    | 0   | Winter | 29642      | 25191    | 0       | 0    |
| 2014-01-08 | 1     | Wednesday | False         | 0.9     | -2.6    | -0.5     | 0.0        | 2.66       | 7.9  | 0.8 | 48.0| 0.003| 53.0 | 33.9 | Good     | Good      | Good     | Good       | Moderate   | 97  | Moderate  | 2014  | 13            | 8            | 216.0            | 27.0        | 0   | 0   | 0   | 0   | 0     | 0    | 1   | Winter | 29642      | 25191    | 0       | 0    |
| 2014-01-09 | 1     | Thursday  | False         | 0.1     | -10.3   | -4.4     | 0.0        | 3.92       | 8.9  | 0.7 | 42.0| 0.021| 46.0 | 18.0 | Good     | Good      | Good     | Good       | Moderate   | 63  | Moderate  | 2014  | 3             | 3            | 350.0            | 116.67      | 0   | 0   | 0   | 0   | 1     | 0    | 0   | Winter | 29642      | 25191    | 0       | 0    |

For a more complete table description, see [https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/README.md](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/README.md).

## Data Collection Ethical Analysis

Since local and historical weather data [2] was obtained through public API usage, we can be confident that no data privacy laws are being violated through collection of this data. As for enrollment and calendar archive data [5] [6], although not publicly accessible data, was granted access for use by my university's archivist. Since air quality data was obtained through a public government agency where they posted their historical records, we can be confident that using this data is ethical for the purposes of this analysis. The data in particular is most likely reliable since it was collected from a federal agency.

The traffic citations data set was the only data set that was scraped on a large scale. This may raise some ethical concerns. However, given that this data was accessible to any user with BYU credentials it seemed reasonable to proceed with the data collection. Furthermore, I encrypted the license plate/vin number in my final data set to prevent linkage with external data sets containing these same plate numbers. Upon checking the *robots.txt* I found no policy against scraping. I further searched the terms of use, and while I didn't find anything that mentioned the action of scraping data from BYU, it states that users will agree to, "not deliberately perform an act that may seriously impact the operation of university systems and resources...[and] not deliberately perform acts that compromise or monopolize resources to the exclusion of others." When scraping large data sets--especially in the manner that I did: sending thousands over requests over the span of several of hours--this could take up resources from the server that could prevent others from accessing BYU. I didn't notice any significant difference in server response speed when ran my scraper, however, one solution to this could be to increase the interval between requests that I send to the BYU server to decrease the amount of resources I'm using.

## Concluding Statement

I curated a final data set, *Provo.csv* using traffic citations scraped from BYU's parking citations server [1], merged with local and historical weather data [2] and historical air quality for each day [3]. Additionally, to introduce a set of controls, I merged this data set with changes in enrollment and holiday changes. I used methods of web scraping, API requests, and PDF extraction to extract this data from their sources and compile them into a single data set for exploratory data analysis and regression analysis. With this data set we hope to determine wether changes in weather and air quality distinctly change the incentives of students to change their mode of transportation. We considered the ethical ramifications of using the data I collected. Overall, given that most of the data originate from credible sources, and given that we are either using publicly available data or have taken precautions to use the data, we can proceed with our analysis.

## Data Sources

[1] University traffic citations data come from BYU's citations server: [https://cars.byu.edu/citations](https://cars.byu.edu/citations). Data obtained through web scraping techniques which I explain [here](https://samleebyu.github.io/2023/09/29/selenium-best-practices/). Raw data can be viewed [here](https://github.com/SamLeeBYU/BYUTrafficCitations/blob/main/ParkingCitationsEncrypted.csv), though license plate/vin numbers have been encrypted so the data set cannot be easily merged with other data sets containing these license plate/vin numbers.

[2] Local and historical weather data for Provo, UT were obtained through the climate API, [Open-Meteo](https://open-meteo.com/en/docs/climate-api).

[3] Local historical air quality data containing measurements for $CO$, $NO_2$, $O_3$, $PM 10$, and $PM 2.5$ were obtained through parsing through data on the Utah Department of Environmental Quality's [website](https://air.utah.gov/dataarchive/archall.htm). Missing data were corrected by substituting the missing values for the average of the given metric for the corresponding month aggregated overall all the data (years 2014-2022, April-August of 2020 excluded).

[4] Air quality pollutant specific sub-category metrics are determined by the U.S. Environmental Protection Agency Office of Air Quality Planning and Standards Air Quality Assessment Division. See [here](https://airnowtomed.app.cloud.gov/sites/default/files/2020-05/aqi-technical-assistance-document-sept2018.pdf) (pages 4 and 5). Appropriate AQI was also calculated using EPA's documentation found here.

[5] Past enrollment data of BYU for the past ten years was obtained curtesy of [BYU Research & Reporting Enrollment Services](https://tableau.byu.edu/#/site/BYUCommunity/views/UniversityEnrollmentStatistics/EnrollmentStatistics).

[6] BYU academic archive data were found courtesy of the HBLL. Records were used to link up when classes started and ended for each semester. Archives can be found [here](https://lib.byu.edu/collections/byu-history/).