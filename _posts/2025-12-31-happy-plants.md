---
layout: post
title: "Happy Plants: A data-driven predictive watering notification system"
date: 2025-12-31
description: In this project, we will create This metadata description may be displayed by search engines, so ensure it entices potential viewers. Buy buy buy!
img: happy-plants/robots.png
tags: [Project] # Personal, Opinion, Technical, Review, Project, Testing
main_page_summary: "Building a system that sends me a daily email to tell me if I should water my plants." 
---

## Executive Summary

In this post I give an overview of the Happy Plants system that I built to deliver me a daily email to tell me if I should water my plants. This involved:
* Python as the core programming language, with Jupyter notebooks used for prototyping
* Scraping data from the web using the BeautifulSoup and requests libraries
* Manipulating data using the Pandas library and storing the data in an Sqlite database
* Sending emails using the Google Gmail API, including generating and including inline data visualisations
* Daily automation using Windows Task Scheduler

The code is available at [https://github.com/alextgould/happy-plants](https://github.com/alextgould/happy-plants).

## Table of Contents

- [Introduction](#introduction)
- [Help a project out? Spare some rainfall data?](#help-a-project-out-spare-some-rainfall-data)
- [Let me send a pretty email? Pretty please?](#let-me-send-a-pretty-email-pretty-please)
- [I'm wicked and I'm lazy, just do it for me daily](#im-wicked-and-im-lazy-just-do-it-for-me-daily)


## Introduction

This is the first in a series of posts relating to the Happy Plants data-driven predictive watering notification system. In this first post, we'll create the system in our local Windows environment, with automation using the Windows Task Scheduler.

In subsequent posts, we can extend this system in various ways, such as:
* Deploy the system to a cloud platform (GCP and/or AWS), using a Linux environment with automation using cron/systemd. This could also involve using Terraform to create a reproducable infrastructure model that others can use to deploy the system.

* Explore alternative models for making the watering recommendation, such as reinforcement learning and/or machine learning models. This could also involve creating synthetic data to train the models on (noting that there isn't a history of forecasts, only of actual rainfall).

* Create an API to register when the user waters their plants, which can be accessed by clicking a button in the daily email. This way the system doesn't have to assume that the user has watered their plants, and the user can ignore the recommendation and get a reminder the next day.

For now, let's focus on getting the basic system up and running. There's lots to do so let's get stuck into it!

## Help a project out? Spare some rainfall data?

I started by Googling to see if there was an existing source of rainfall data for my region. I did find a few R packages where people had looked at sourcing the data, but I'm using Python, and in any case one of the packages had "left town" with a comment about "BOM's ongoing unwillingness to allow programmatic access to their data and actively blocking any attempts made using this package or other similar efforts" which didn't sound promising. After poking around on the BOM FTP Server, I decided to keep things simple and use the rainfall data from the public sites. This comes from two sources:

* [http://www.bom.gov.au/nsw/forecasts/sydney.shtml](http://www.bom.gov.au/nsw/forecasts/sydney.shtml) - contains daily forecasts for the coming week, with % chance and possible rainfall range if the chance exceeds some threshold

* [http://www.bom.gov.au/jsp/ncc/cdio/weatherData/av?p_nccObsCode=136&p_display_type=dailyDataFile&p_stn_num=66037](http://www.bom.gov.au/jsp/ncc/cdio/weatherData/av?p_nccObsCode=136&p_display_type=dailyDataFile&p_stn_num=66037) - contains historical rainfall measurements taken at Sydney Airport AMO (station number 066037)

I was impressed that ChatGPT was able to shed some light on the query parameters needed for the historical data; for example, p_nccObsCode specifies the type of observation data to retrieve with 136 retrieving daily rainfall data while 139 retrieves daily temperature data. ChatGPT also guessed that the less documented parameter p_c might be a session identifier, a fact that I was later able to confirm through trial and error. Passing an incorrect p_c value will result in an error page. Luckily, passing no p_c value gives you the data for the current year without any further effort. To obtain data for prior years, you would need to first obtain a p_c value and then use this to obtain the prior year data.

The first function I created uses requests to obtain the page source:

```python
def _get_page_source(url):
    """Return page source from url (pretending to be a Chrome browser)"""

    headers = {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36'
    }

    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        return response.text
    else:
        raise Exception(f"Failed to fetch page: {response.status_code}")
```

Without the User-Agent value in the headers, we get a 403 Forbidden error. You can get an appropriate header value using the Developer Tools in Chrome (F12 > Network > F5 to refresh > click the html file and look for User-Agent value under Request Headers). Once we pass headers to make the API think a web browser is making the request, rather than the requests library, we get our data in raw html format.

Once we have the html, we can look for patterns that beautiful soup can use to extract the data. Using `print(soup.prettify())` in a Jupyter notebook I spot this recurring pattern:

``` html
<div class="day">
<h2>
  Friday 21 March
</h2>
<div class="forecast">
  <dl>
  <dt>
    Summary
  </dt>
  <dd class="image">
    <img alt="" height="42" src="/images/symbols/large/showers.png" width="45"/>
  </dd>
  <dd>
    Min
    <em class="min">
    20
    </em>
  </dd>
  <dd>
    Max
    <em class="max">
    29
    </em>
  </dd>
  <dd class="summary">
    Shower or two.
  </dd>
  <dd class="rain">
    Possible rainfall:
    <em class="rain">
    0 to 3 mm
    </em>
  </dd>
  <dd class="rain">
    Chance of any rain:
    <em class="pop">
    50%
    <img alt="" height="10" src="/images/ui/weather/rain_50.gif" width="69"/>
    </em>
  </dd>
  </dl>
  <h3>
  Sydney area
  </h3>
  <p>
  Partly cloudy. Medium chance of showers. Light winds.
  </p>
</div>
<p class="alert">
  Sun protection recommended from  9:40 am to  4:20 pm, UV Index predicted to reach 8 [Very High]
</p>
</div>
```

To convert this into a dataset, I used Jupyter notebooks where I could iteratively make incremental changes and immediately see the results.

I also leveraged ChatGPT to set out the basic code, based on simple examples of the html and desired output, as well as quickly change things that weren't quite right. This was actually a common theme in this project; while in the past my superpower would have been systematically and stubbornly figuring out how to do these things, increasingly I'm adding value by being broadly aware of what needs to be done, framing the question in the right way to get the correct answer out of ChatGPT/Copilot, and reviewing its outputs to identify anything that needs to be improved. It's not that I couldn't do things manually, it's that it's so much quicker, particularly when using packages or languages that you only use occasionally rather than on a daily basis.

Here's the final function used to extract the forecast data:

```python
def _extract_forecast_data(soup):
    """Extract rainfall chance and amounts from the Beautiful Soup class"""

    sections = soup.find_all(class_="day")
    results = []
    for section in sections:

        # extract the date (which is implied by the current date and this being a forecast for the coming week, includes rest of today)
        date_forecast_applies_to = section.find('h2').text.strip()
        date_forecast_was_made, date_forecast_applies_to = _convert_to_datetime(date_forecast_applies_to)

        rain_section = section.find_next('dd', class_="rain")
        rain_mm_low = 0
        rain_mm_high = 0
        if "Possible rainfall" in rain_section.text: # rainfall mm is only shown when rainfall chance exceeds some threshold
            rain_mm = rain_section.find_next('em', class_="rain").text.strip()
            match = re.search(r"(\d+)\s*to\s*(\d+)", rain_mm) # convert 0 to 3 mm into values 0 and 3 using regular expressions
            if match:
                rain_mm_low = int(match.group(1))
                rain_mm_high = int(match.group(2))

        rain_chance = rain_section.find_next('em', class_="pop").text.strip()
        rain_chance = float(rain_chance.strip('%')) / 100 # convert from text % to float now (might be easier than doing so later on)

        results.append([date_forecast_was_made, date_forecast_applies_to, rain_chance, rain_mm_low, rain_mm_high])

    df = pd.DataFrame(results, columns=['date_forecast_was_made', 'date_forecast_applies_to', 'rain_chance', 'rain_mm_low', 'rain_mm_high'])
    df['date_forecast_was_made'] = pd.to_datetime(df['date_forecast_was_made'], format='%Y-%m-%d')
    df['date_forecast_applies_to'] = pd.to_datetime(df['date_forecast_applies_to'], format='%Y-%m-%d')

    return df
```

We use class_ to find relevant elements in the html, use regular expressions to convert phrases such as "0 to 3 mm" into values, format the values and append them to a list of lists, then convert this into a pandas DataFrame, formatting dates along the way. I followed a similar process to extract the historical data, which had its own unique html structure.

Date formatting was actually something I had to revisit a few times while creating the database. The %Y-%m-%d format (e.g. 2025-03-26) is known as the ISO 8601 standard format and is commonly recognized by most systems. %Y/%m/%d (with slashes) is also a valid date format, but it is less commonly used in ISO 8601-compliant systems. It's more of a regional format, often used in some European and North American contexts, but not the canonical ISO 8601 format. These are the sorts of interesting things you find out when you get curious and go down a rabbit hole with ChatGPT at your side.

We'll revisit the issue of dates shortly when we discuss storing the data in the database and using it for pandas data manipulations.

## Database.py

For pandas calculations, having dates in datetime format yyyy-mm-dd 00:00:00 is useful. I'm undecided if it makes sense to store the 00:00:00, considerations:
  - outputs are more verbose
  - would need to test filtering e.g. does something like "= 'yyyy-mm-dd'" still work or do you need "= 'yyyy-mm-dd 00:00:00'"
  - currently using date as PK so process is idempotent for forecasts within a given day. would likely need to remove prior ones on rerun within a given day, or just fix the time component to 00:00:00 regardless of when it's actually run

## Modelling approach

Approach
* pick up the historical days leading up to the forecast day
* pick up all forecast_applies_to data for the date_forecast_was_made day
* Flatten both of these, using the difference between the date and the forecast date to produce an index
* Then keep the relevant ones (e.g. past X days, next Y days)
* "feature selection" e.g. if the success criteria is 20mm per week, add up the watering for the past 6 days
as this is guaranteed (and includes adjustments for assumed action taken following prior watering notifications)
and/or add up some expected rainfall (e.g. rain_chance * average of rain_mm_low and rain_mm_high, for next 1 day or possibly 
each future day. 
Relevance of this aspect might depend on what model(s) I'm using. In the short term we might go with something super simple
(e.g. just use actual past 6 days + expected for next 2 days and issue notification if <20mm), particularly while
collecting/generating data and setting up notification flow

Assumptions / Areas of uncertainty
* include current day forecast?
  - time of day that the program is run (e.g. running at the start of the day vs at the end of the day)
  - time of day that the forecasts are released (e.g. is this 9am? 6am? 12am? are they updated throughout the day? on an hourly basis?)
  - wording implies it's for the remainder of the day which could be misleading (e.g. it rains all morning, then forecast is 0.0)
  - ideal time of day to run this might depend on when the user is going to take action (e.g. free to water at 7am vs 10am vs 6pm)
* assume that past notifications resulted in watering at the required amount (need to overwrite historical data after extracting it)

Note on the forecast page
http://www.bom.gov.au/nsw/forecasts/sydney.shtml
it actually has e.g. 
"Forecast issued at 4:20 pm EDT on Thursday 20 March 2025."
so looking at this page a few times manually will probably give sufficient info to get a feeling for this
particularly if it's a day where it's been raining and clears up

At 7am, 9am:
Forecast issued at 4:45 am EDT on Friday 21 March 2025.
70% chance of rain with 0 to 7 mm of rain

At 10am:
Forecast updated at 9:29 am EDT on Friday 21 March 2025.
70% chance of any rain, does not show the possible rainfall figures

At 2pm:
Forecast updated at 11:09 am EDT on Friday 21 March 2025.
70% chance of any rain

Might actually be more sensible to include forecast rainfall from yesterday for today, then use forecast rainfall for future days from today (a bit complicated but possibly the most appropriate approach)


## Let me send a pretty email? Pretty please?

The image below shows the emails, first using an attachment, then with the inline images which are much more easily accessible.

![]({{site.baseurl}}/assets/img/happy-plants/email_inline.png)

## I'm wicked and I'm lazy, just do it for me daily

Daily.py....

The repo structure includes the following:

```bash
happy-plants/
│── scripts/               # Scripts that call modules from src folder
│   │── daily_run.py       # Runs daily pipeline
│── src/                   # Main Python code
│   │── create_plots.py    # Create images from data in database for emails
│   │── database.py        # Handles SQLite database interactions
│   │── get_data.py        # Source data from websites
│   │── pred_models.py     # Models that predict whether we should manually water or not
│   │── prepare_data.py    # Data manipulation for predictive models
│   │── send_email.py      # Send emails using Google's Gmail API
```

You would think it would be simple to have daily_run.py call modules from the scripts in the src directory. If you ask ChatGPT, you'll probably be told to include a \_\_init\_\_.py file in the src folder. Well it turns out that advice doesn't apply since 

```python
# Add src to path (rather than having to create .vscode\settings.json, adjust PYTHONPATH in Task Scheduler etc)
import sys
src_path = os.path.abspath(os.path.join(os.path.dirname(__file__), '../src'))
if src_path not in sys.path:
    sys.path.append(src_path)
    logger.debug(f"appending {src_path} to sys.path so that modules in src directory can be imported")
```

TODO: talk about the whole \_\_init\_\_.py misleading ChatGPT advice

With this in place, we can now import ...

```python
# Local imports
from get_data import forecast_data, historical_data
import database, prepare_data, pred_models, send_email, create_plots
```

### Automation using Windows Task Scheduler


And then finally... use transitions (spoiler alert: we're almost done)

## But wait, there's more!

If you've made it this far, you'll surely enjoy a few bonus topics that came up along the way. 

Along the way, I picked up a few things that I think are interesting to talk about...

### Logging

```python
# Logging setup (specifying file here means you don't need to redirect log outputs when setting up Task Scheduler etc)
import logging
import os
log_file = os.path.join(os.path.dirname(__file__), f"{os.path.splitext(os.path.basename(__file__))[0]}.log") # e.g. collect_data.py -> collect_data.log
logging.basicConfig(filename=log_file, level=logging.INFO, format="%(asctime)s %(levelname)s: %(message)s", datefmt="%Y-%m-%d %H:%M:%S")
logger = logging.getLogger(__name__)
logger.debug(f"log_file being written to {log_file}")
logging.getLogger('matplotlib').setLevel(logging.WARNING)
logging.getLogger('PIL').setLevel(logging.WARNING)
```



### Snippets

F1 > Snippets > Configure snippets e.g.

```json
{
  "Logger Setup": {
    "prefix": "logger",
    "body": ["# Logging setup",
      "import logging",
      "import os",
      "log_file = os.path.join(os.path.dirname(__file__), f\"{os.path.splitext(os.path.basename(__file__))[0]}.log\") # e.g. collect_data.py -> collect_data.log",
      "logging.basicConfig(filename=log_file, level=logging.INFO, format=\"%(asctime)s %(levelname)s: %(message)s\", datefmt=\"%Y-%m-%d %H:%M:%S\")",
      "logger = logging.getLogger(__name__)"],
    "description": "Logging setup for the top of a script. Consider removing basicConfig if it's run elsewhere. Consider removing log_file if you don't want to redirect the log output to a file."
  },

  "Log Debug Statement": {
    "prefix": "log",
    "body": "logger.debug(f\"$1 {}\")",
    "description": "Insert a logger.debug statement"
  },

  "Main Function": {
    "prefix": "main",
    "body": [
      "def main():",
      "\t$1",
      "",
      "if __name__ == \"__main__\":",
      "\tmain()"
    ],
    "description": "Insert the __name__ check and main function often found at the bottom of python files"
  },

  "Project Root Global Variable": {
    "prefix": "root",
    "body": ["# Paths relative to project root directory",
      "import os",
      "PROJECT_ROOT = os.path.abspath(os.path.join(os.path.dirname(__file__), \"..\"))",
      "image_path = os.path.join(PROJECT_ROOT, 'img', \"my_image.png\")"
    ],
    "description": "Create a global variable called PROJECT ROOT that can be used for more robust relative referencing"
  }
}
```

### Docstrings



## Conclusion

Wrap up and reinforce the main takeaway (probably Thai), key points, next steps, call to action etc.
