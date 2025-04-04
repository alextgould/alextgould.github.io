---
layout: post
title: "Happy Plants: A data-driven predictive watering notification system"
date: 2025-03-31
description: In this project, we will create This metadata description may be displayed by search engines, so ensure it entices potential viewers. Buy buy buy!
img: happy-plants/robots.png
tags: [Project] # Personal, Opinion, Technical, Review, Project, Testing
main_page_summary: "Building a system that sends me a daily email to tell me if I should water my plants" 
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
- [Getting rainfall data is easy, right?](#getting-rainfall-data-is-easy-right)
- [Data warehouse - abstractions are just the beginning](#data-warehouse---abstractions-are-just-the-beginning)
- [My modelling approach - filthy and gorgeous](#my-modelling-approach---filthy-and-gorgeous)
- [Let me send a pretty email? Pretty please?](#let-me-send-a-pretty-email-pretty-please)
- [Autobots, roll out!](#autobots-roll-out)
- [Me want cookie! Om nom nom nom](#me-want-cookie-om-nom-nom-nom)
- [Conclusion](#conclusion)

## Introduction

This is the first in a series of posts relating to the Happy Plants data-driven predictive watering notification system. In this first post, we'll create the system in our local Windows environment, with automation using the Windows Task Scheduler. There's lots to do, so let's get stuck into it!

## Getting rainfall data is easy, right?

I started by Googling to see if there was an existing source of rainfall data for my region. I found some R packages where people had attempted this, but I'm using Python, and in any case, it seemed like things weren't going all that well:

> "This package has been archived due to BOM's ongoing unwillingness to allow programmatic access to their data and actively blocking any attempts made using this package or other similar efforts."
> 
> \- disgruntled R package author

This didn't sound promising. After poking around on the BOM FTP Server, I decided it might be more entertaining and instructive to scrape the rainfall data from the website.

The data comes from two sources:

* [http://www.bom.gov.au/nsw/forecasts/sydney.shtml](http://www.bom.gov.au/nsw/forecasts/sydney.shtml) - contains daily forecasts for the coming week, with rainfall chance and a minimum-maximum range for possible rainfall amount

* [http://www.bom.gov.au/jsp/ncc/cdio/weatherData/av?p_nccObsCode=136&p_display_type=dailyDataFile&p_stn_num=66037](http://www.bom.gov.au/jsp/ncc/cdio/weatherData/av?p_nccObsCode=136&p_display_type=dailyDataFile&p_stn_num=66037) - contains historical rainfall measurements taken at Sydney Airport AMO (station number 066037)

I won't go into all the details of the historical rainfall API, but one interesting feature I found was the p_c query parameter which gets added when you first visit the page. This is a session value which you need to pass for subsequent queries or the site will reject your request. We would need to use this, for example, if we wanted to obtain data from prior years. Luckily, we only need the current year data, which is what gets displayed if you omit the p_c and other query parameters.

The first Python function I created uses the requests library to obtain the page source code:

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

The function includes a User-Agent value in the header, without which we get a 403 Forbidden error. You can get an appropriate header value using the Developer Tools in Chrome (F12 > Network > F5 to refresh > click the html file and look for User-Agent value under Request Headers). Passing the header value makes the API think a web browser is making the request, which gives us our data in raw html format.

We can then pass the html from this function to the [Beautiful Soup](https://beautiful-soup-4.readthedocs.io/en/latest/) package. This package represents the html as a nested data structure which is easier to navigate and extract data from:

```python
from bs4 import BeautifulSoup
url = "http://www.bom.gov.au/nsw/forecasts/sydney.shtml"
src = get_page_source(url)
soup = BeautifulSoup(src, 'html.parser') # recommended to include html.parser here to ensure consistent cross-platform results
```

The forecast page looks like this:

![]({{site.baseurl}}/assets/img/happy-plants/forecast_web.png)

Using `print(soup.prettify())` in a Jupyter notebook, we can review the underlying html and identify the patterns that we'll use to extract the data. I found it useful to work in a Jupyter notebook as I could iteratively make incremental changes and immediately see the results. I also found it useful to copy a segment of the html and remove all the elements that weren't needed so it's easier to identify the important elements:

``` html
<div class="day">
  <h2>
    Saturday 29 March
  </h2>
  <div class="forecast">
    <dd class="rain">
      Possible rainfall:
      <em class="rain">
      20 to 70 mm
      </em>
    </dd>
    <dd class="rain">
      Chance of any rain:
      <em class="pop">
      95%
      <img alt="" height="10" src="/images/ui/weather/rain_95.gif" width="69"/>
      </em>
    </dd>
    </dl>
  </div>
</div>
```

As an aside, giving this sort of simple input to ChatGPT along with some simple target outputs makes it easy to quickly get some initial BeautifulSoup code to refine. This was actually a common theme in this project; while in the past my role would have been systematically and stubbornly figuring out how to do everything myself, increasingly I'm adding value by being broadly aware of what needs to be done and how to do it, framing the question in the right way to get a fairly accurate initial response out of ChatGPT or Copilot, reviewing its outputs to identify anything that needs to be improved, and then making the final adjustments myself or in conjunction with it. It's not that I couldn't do things myself manually, it's just that it's so much quicker, particularly when using packages or languages that you only use every so often rather than on a daily basis. Essentially, it feels like being a surgeon or a manager with staff supporting you, except you don't have to organise multiple meetings to get things done.

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

We use class_ to find the relevant sections in the html, use regular expressions to convert phrases such as "0 to 3 mm" into values, format the values and append them to a list of lists, then convert this into a pandas DataFrame, formatting dates along the way.

I followed a similar process to extract the historical data, which had its own unique html structure. In particular, the historical data is presented in a table with days of the month on the left side (1st 2nd 3rd...) and months on the top (Jan Feb Mar...). This required a loop which used regular expressions to convert the days into integers (1 2 3...), extract data for each month, create proper dates including the year at the top of the table, and place this all in a dataframe. 

We have our rainfall data, but now we need to store it somewhere.

## Data warehouse - abstractions are just the beginning

![]({{site.baseurl}}/assets/img/happy-plants/data_warehouse.jpg)

(I had to do something dramatic to make "ETL" more exciting)

I decided to use a sqlite database, because it's a lightweight and self-contained database, with minimal setup and administration, which has maintained a steady level of popularity within recent years and is likely to be relevant into the future. A lot of my experience to date has been in enterprise-level applications using PostgreSQL or MSSQL, where concurrency needs, high availability or security requirements were more critical. It was refreshing to be able to simply create the file and start using it.

I developed a database script to handle interactions with the database, so this aspect can be abstracted away, with my other scripts simply calling a function to add data to the database or get data from the database. For example, in functions that get data, an optional parameter can be passed to filter data using standard SQL WHERE syntax.

Similar to the web scraping exercise, I found it useful to prototype using a Jupyter notebook, so I could visualise the data and confirm things were working as intended. This was particularly the case with respect to dates.

### It's always the Date fields that need special attention

My vision for the project was that it would be run on a daily basis, but I wanted to ensure the process was idempotent i.e. if the data collection script is run multiple times in the same day, only the latest values are kept. To achieve this, I included the date fields within the primary key, saved as TEXT in a yyyy-mm-dd format. Saving the data in this format also means that examining and filtering data in the database is cleaner, as there isn't an unnecessary 00:00:00 at the end of the date as would be the case if a datetime was stored. The code below gives an example of how this was done:

```python
date = datetime(int(year), month_number + 1, day_number).strftime('%Y-%m-%d') # add year and format as yyyymmdd
```

The %Y-%m-%d format (e.g. 2025-03-26) is known as the ISO 8601 standard format and is commonly recognized by most systems. %Y/%m/%d (with slashes) is also a valid date format, but it is less commonly used in ISO 8601-compliant systems. It's more of a regional format, often used in some European and North American contexts, but not the canonical ISO 8601 format. These are the sorts of interesting things you find out when you get curious and go down a rabbit hole with ChatGPT at your side.

While the data is saved in the database as TEXT, it's helpful to have it formatted as pandas datetime when we extract it, as this is required for pandas to do things like adding days to dates or calculating the number of days between two dates. Hence, in the functions that extract data from the database into a python DataFrame, we convert it to datetime, but still format it as ISO 8601. This means when we display the dates they'll still look like yyyy-mm-dd:

```python
df['date'] = pd.to_datetime(df['date'], format='%Y-%m-%d')
```
### Making the database more classy

Early on, I had the bright idea that someone (my future self perhaps) might want to change the location of the database being used. Perhaps multiple databases would be used at once, so passing the database location to each of my functions seems like a good idea? Oh, and we need some code that connects to the database and/or checks that a database exists at that location. This code is being used in every function, so let's put it in a function and call that function from our other functions. What a mess!

To make things cleaner, I decided to create a class that would allow me to pass the database location if I didn't want to use the default one. It also had a helper function to connect to the database, which would be useful if I decided to swap from sqlite to another database, perhaps as part of deploying the system to a cloud provider. The end product looked something like this:

```python
class RainfallDatabase:
    def __init__(self, db_path=DEFAULT_DB_PATH):
        """Initialize database with a given path."""
        self.db_path = db_path
        if not os.path.exists(self.db_path):
            raise FileNotFoundError(f"Database file does not exist at {self.db_path}")

    def _connect(self):
        """Helper function to create a database connection."""
        return sqlite3.connect(self.db_path)
    
    # Functions to create (or reset) database tables e.g.

    def create_historical_table(self, reset=False):
        """Create the historical table, with an option to reset it."""
        with self._connect() as conn:
            cur = conn.cursor()
            if reset:
                logger.info("Resetting historical table")
                cur.execute("DROP TABLE IF EXISTS historical;")
            cur.execute("""
            CREATE TABLE IF NOT EXISTS historical (
                date TEXT PRIMARY KEY,
                rainfall_mm REAL
            );
            """)
            conn.commit()
            logger.info("Historical table created or verified successfully.")
```
In hindsight, I might have committed to using a single database, with the location saved in a config file, and not used classes at all. It is a minor inconvenience to have to instantiate the class each time you want to interact with the database:

```python
# save the data to the database    
db = database.RainfallDatabase()
db.add_historical_data(df_historical)
```

This highlights an aspect of coding that isn't often talked about - while code can seem fairly deterministic, there's often a lot of 'art' in the design decisions being made, with more than one way of solving any given problem and pros and cons to each approach.

### Avoiding SQL injection

As a final insight into this stage of the process, I had a good chat with my "bestie" (ChatGPT) about the use of placeholders in my add data functions. For example:

```python
def add_preds_data(self, model, date, pred):
    """Add preds data to the database."""
    with self._connect() as conn:
        cur = conn.cursor()
        cur.execute("""
            INSERT OR REPLACE INTO preds (model, date, pred)
            VALUES (?, ?, ?)
            """, (model, date, pred))
        conn.commit()
```

Here we use a parameterised query with placeholders (?) to avoid SQL injection - the risk that an attacker can pass a value that completes the current query, then runs another malicious query. This is more important when dealing with user input, but is still good practice in any situtation. It's safer and more efficient then directly inserting values with format strings:

```python
f"INSERT INTO table VALUES ({value1}, {value2})"
```

With the parameterised approach, SQLite handles the escaping of values for us (so the value passed is treated as a complete value and not part-value part-additional malicious query), along with making our code more readable and efficient.

Interestingly, moving to PostgreSQL would require a slightly different syntax (using $1 $2 instead of ? ?, as well as ON CONFLICT DO UPDATE SET instead of INSERT OR REPLACE). Hence, it wouldn't be enough just to update our _connect function at the top of the class, we would actually have to review some of our queries also.

In any case, we now have our data in a database, and it's time to start thinking about data transformations and our approach to getting predictions on whether we should manually water our plants or not.

## My modelling approach - filthy and gorgeous

### Assumptions

Before we prepare the data for modelling (or perhaps while doing so if we lack foresight), we need to make a few assumptions:

* **Weekly watering targets** - based on a robust process of asking ChatGPT how much I should water my hedge and how often, we go with a target of 20mm of water over the course of a week. In reality, different plants need different amounts of water at different watering frequencies. A fully fledged product might let the user list out all their plants, how old they are, what soil they're in and so on. The real world is complex, but for my toy example, we just need 20mm of water each week.

* **Laziness factor** - If our model tells us to manually water, and we water our plants, we don't want the model to keep telling us to water our plants every day for the next week just because it hasn't rained. But what if we're lazy and we ignore its advice? Perhaps then we do want a reminder the next day. A fully fledged product might let the user click a button to notify the system that they followed its advice. For my toy example, we assume that the user always follows the guidance to manually water.

* **Manual water amount** - I have a drip hose that I can turn on to water my hedge, and when use it, I generally leave it on for about 30 minutes. A fully fledged product might let the user specify their watering system, house water pressure, time of day that they are watering etc, then do the calcs and provide tailored recommendations on how much to water, recommending less when there's been some rain but not enough over recent days. For my toy example, we assume that the user gives the plants their target water amount and you can never have too much water.

* **Manual water timing** - When I water, I usually try to do so before 10am. My layman’s understanding is that watering in the middle of the day is less effective as the water evaporates, and having water on the leaves in the sun can burn them, but watering late in the day can be less healthy for the plant if the water on the leaves fails to evaporate. The historical data is nominally measured at 9am for the prior 24 hours, so running the model at around 10am might give the most up-to-date data for making predictions, but the recommendation could also be made too late to be of any use. For my toy example, we assume the user generally waters early in the day, so we'll collect the data at 7:30am.

* **Current day forecasts** - The forecast data includes a forecast for the current day, which you might think would be quite useful, depending on when the user typically waters their plants and when the data collection process is run. However, it says something like "Forecast for the *rest* of Friday", which might mean it changes in unexpected ways over the course of the day (and in actual fact, forecast amounts are actually removed later in the day, rendering it particularly useless). To avoid this uncertainty and keep things simple, for my toy example I base today's data on yesterday's forecast, so it doesn't matter at what point in the day the process is run, the prediction will always be the same.

Having spent some time setting out some gorgeous assumptions, we can go ahead and transform the data in order to make predictions.

### Data transformations

The data is currently stored in tables with one row per date. We need to transform this into rows of data with all the predictive values our model needs for any given day. This primarily includes the last week of historical rainfall and the latest rainfall forecasts.

The first step is to filter the relevant data, using today's date, pandas timedelta and the filter functionality of our database functions. For example:

```python
# start with today's date

if forecast_date == "":
    forecast_date = datetime.today().date().strftime('%Y-%m-%d') # e.g. '2025-03-21'
forecast_dt = datetime.strptime(forecast_date, '%Y-%m-%d')

# get date cut offs in ISO 8601 format

end_dt = forecast_dt + timedelta(days=forecast_days)
end_date = end_dt.strftime('%Y-%m-%d')

# filter relevant data

db_forecast_filter = f"date_forecast_was_made='{forecast_date}' AND date_forecast_applies_to > '{forecast_date}' AND date_forecast_applies_to <= '{end_date}'"
df_forecast = db.get_forecast_data(filter=db_forecast_filter)
```

A similar approach is used for the historical data, but taking into account the history of model predictions and replacing the historical rainfall with the target amount when the model tells us to manually water. This is a simple way of ensuring the model doesn't keep recommending we water each day for the next week.

With the relevant data sourced, we just need to add some indices and merge it all together:

```python
# add indices based on date differences
df_historical['hist_index'] = (forecast_dt - df_historical['date']).dt.days
df_forecast['forecast_index'] = (df_forecast['date_forecast_applies_to'] - forecast_dt).dt.days

# where the model recommended watering, assume this was done at the default_water_mm
df_historical.loc[df_historical['date'].isin(df_preds['date']), 'rainfall_mm'] = DEFAULT_WATER_MM

# pivot data into columns in dict
hist_mm = {f'hist_{row.hist_index}': row.rainfall_mm for _, row in df_historical.iterrows()}
forecast_chance = {f'chance_{row.forecast_index}': row.rain_chance for _, row in df_forecast.iterrows()}
forecast_mm = {f'mm_{row.forecast_index}': row.rain_mm_high for _, row in df_forecast.iterrows()}

# add current day forecast
if not df_current_day_forecast.empty:
    forecast_chance['chance_0'] = float(df_current_day_forecast.iloc[0]['rain_chance'])
    forecast_mm['mm_0'] = float(df_current_day_forecast.iloc[0]['rain_mm_high'])

# merge all data
model_data = {'date': forecast_date, **hist_mm, **forecast_chance, **forecast_mm}
col_order = ['date'] + sorted(hist_mm.keys()) + sorted(forecast_chance.keys()) + sorted(forecast_mm.keys())
df = pd.DataFrame([model_data], columns=col_order)
df['date'] = pd.to_datetime(df['date'], format='%Y-%m-%d')
```

When creating the indices, we see the value in having our dates pre-formatted as pandas datetime values, so we can use .dt.days.

I'll confess I let ChatGPT create my list comprehensions for the pivot step in the middle. As mentioned previously, knowing what you want done and letting it handle the finer details often leads to a fairly elegant solution fairly quickly. Another example of this came up when creating the target dataset to train a machine learning model, where the target is based on the actual rainfall from the past 7 days:

```python
df_historical["rainfall_mm_week"] = df_historical["rainfall_mm"].rolling(window=7, min_periods=7).sum()
```

Having this one liner pop out without having to check (or be familiar with) the .rolling syntax is much quicker than having to look it up (or be intimately familiar with it).

When merging the data at the end, using col_order allows us to place our columns in a desired order of date > historical > forecast, despite having separately added in the current day forecast values with a _0 index.

This process gives us a single row of data that a model can use to make a prediction. This will be used on a daily basis, to generate a model prediction. We can also repeat the process for each day that we've collected forecast data, and combine it with the target values discussed above to generate a training dataset for a machine learning model.

### Keep it simple

For the purpose of getting something up and running, I set aside my aspirations of building a fancy reinforcement learning model, or even a relatively basic machine learning model, and instead went with a simple logic based approach:

> Add up the actual rainfall from the past six days. If the chance of rain is 50% or more, add on the forecast (maximum) rainfall for today. If the total amount is less than my minimum weekly rainfall target, recommend manual watering today.

I suspect this simple approach is actually most of what you need from a data driven recommendation system. You could even make things even simpler and just use the past week of actual rain data. There's also no history of forecast data, so to train a more sophistocated model we would have to wait around while we build up a history of forecast data (or use synthetic data). All things considered, it makes sense to use our simple logic-driven approach for now.

As a thought exercise, it is interesting to come up with think of a few reasons for why we might consider testing a more sophistocated approach in the future:

* If my plants haven't had water in 6 days and it's not likely to rain today, but it's going to bucket down tomorrow, I'm probably not watering my plants. A reinforcement learning model can take into account my level of laziness, which could be different from one person to the next. Based on a laziness factor, the model can make the judgement call on our behalf.

* A regular machine learning algorithm won't cater for my laziness factor, but it might be able to figure out if the people at the BOM are systematically overstating the chance of rain to avoid the risk that people are grumpy if it ends up raining when they say it's unlikely. In this case, the model might require a 60% chance of rainfall before it includes the forecast rainfall for today.

* Given the minimum rainfall amount is almost always zero, I decided to use the maximum rainfall amount. Perhaps I should be using the average of the two figures, or some other function? We don't fully understand the data generation process (i.e. how the BOM do their forecasting). A machine learning approach could potentially resolve this for us using the data.

I suspect these factors won't actually be that important, so again, it's fine to keep things simple on the mondelling front, and instead focus our attention on sending email notifications.

## Let me send a pretty email? Pretty please?

It took a little while to decide on an approach for sending email notifications. There are many possible approaches one can take, and I knew little about any of them. It was like being a first home buyer, trying to decide what to prioritise without knowing what you don't know:

* How easy will it be for me to get it working in a local/Windows environment?
* How easy will it be for me to migrate it to a cloud/Linux environment?
* How easy it would be for someone else to get it working, if they wanted to grab my repo and take it for a test drive?
* Will it compromise the security of my Google account, or that of others who want to follow in my footsteps?
* What's the upfront and ongoing cost?

It took me three attempts to come up with a solution I was happy with.

### Attempt 1: Third-Party Transactional Email Service

My first attempt was using a third party transactional email service ([SendGrid](https://sendgrid.com/en-us)). It seemed like an easy option - for both myself and others - and it had a free tier. However, it turned out this approach required a lot of details, including multiple email/mobile/address details/verifications, creating sender IDs etc. This was probably a result of the global nature of the service, anti-spam legislative requirements, and the acquisition of SendGrid by Twilio in 2019, and 6 years later they still seem to have multiple sign up processes between them.

To make matters worse, when I went to use my (newly minted "throwaway") Gmail account, I learned that the approach was likely to fail anyway, due to [DMARC](https://www.twilio.com/docs/sendgrid/ui/sending-email/dmarc):

> Gmail, or any other receiving email server, has no way of knowing whether you are using SendGrid to send email for legitimate purposes

Hence, I would need to own and authenticate my own domain just to use this third party service effectively! At this point I realised using a transactional email service wasn't the silver bullet I thought it was, and was more akin to using Azure. There's probably a value proposition there, but it's more about applications at scale and you probably need a decent amount of knowledge to unlock it.

### Attempt 2: Gmail SMTP with App passwords

My second attempt was using Gmail more directly with App passwords. I actually started my journey looking at [this post](https://realpython.com/python-send-email/), which was the first thing that popped up on Google, but I later realised (only from looking at the dates in the comments) that it's 6 years old, which explains why it suggests using the now depreciated "allow less secure apps" method. The modern equivalent is "[App Passwords](https://support.google.com/accounts/answer/185833?hl=en)", which are 16 digit passcodes that allow an app to use your Gmail account. This seemed like an easy and promising solution, but I found the prominent warning a bit unnerving:

> Important: App passwords aren’t recommended and are unnecessary in most cases. To help keep your account secure, use "Sign in with Google" to connect apps to your Google Account.

Despite the warning, I gave it a shot, without success. It turns out there are actually quite a few restrictions for using App passwords, beyond having to have set up 2-Step verification on your Google account. I suspect the issue may have been that I was "logged into a work, school, or another organization account", as my daughter has a Google-based login for her school. I also uncovered limitations such as the App password being revoked any time you change your password. All of this left me feeling once again that this wasn't the silver bullet I was looking for. 

### Attempt 3: Gmail SMTP with OAuth2

Third time's a charm. This time I went with the OAuth2 ("Sign in with Google") approach. The main downside of this approach is that it requires creating a Google Cloud Project, which I initially thought might be a bit advanced or burdensome, but was actually relatively simple (certainly compared to my next best option which was to set up my own mail server, dealing with SPF, SKIM, DMARC and other suspicious acronyms).

The three steps I followed were:

Step 1: Create a Google Cloud Project
  - Go to [Google Cloud Console](https://console.cloud.google.com)
  - (optional) Swap to a new Gmail account you have created (top right)
  - Select a project (top left; will be set to the last active project or "Select a project" if you don't have any existing ones) > "New Project" > add a name (e.g. "Gmail SMTP OAuth2") > Create

Step 2: Enable the Gmail API
  - (select the project) > APIs & Services > Library > Gmail API (under "Google Workspace or use the search box) > Enable

Step 3: Create OAuth 2.0 Credentials
  - APIs & Services > Credentials
  - (if prompted) Configure Consent Screen > Get started
    - App Information: Fill in app name (e.g. "Alex Gould's blog") and user support email (your new Gmail address)
    - Audience: External
    - Contact Information: your new Gmail address
  - Create OAuth client > Application Type: "Desktop App" > Name (e.g. "happy-plants") > Create > Download JSON (save as credentials.json in project folder)

At this point we have a credentials.json file, which contains a client_id and client_secret to authenticate our app to Google, as well as the URIs used during the OAuth2 flow. I believe you can use this file with a third party SMTP Relay service as another option, but creating our own script will give us more flexibility, less limitations and cost, and be more fun.

The following code, based on [the docs](https://developers.google.com/workspace/gmail/api/quickstart/python), handles the authentication:

```python
from google.auth.transport.requests import Request  # Refresh OAuth2 tokens
from google.oauth2.credentials import Credentials  # Handle OAuth2 credentials
from google_auth_oauthlib.flow import InstalledAppFlow  # OAuth2 authentication flow

SCOPES = ["https://mail.google.com/"]  # Full access to send emails

def _get_credentials() -> Credentials:
    """Obtain and refresh OAuth2 credentials as needed."""

    creds = None
    
    # Load existing credentials if available
    if os.path.exists(TOKEN_FILE):
        creds = Credentials.from_authorized_user_file(TOKEN_FILE, SCOPES)
    
    # Refresh or obtain new credentials if needed
    if not creds or not creds.valid:
        try:
            if creds and creds.expired and creds.refresh_token:
                creds.refresh(Request())
            else:
                raise Exception("No valid refresh token available.")
        except Exception as e:
            logger.warning(f"Failed to refresh token: {e}. Re-authenticating...")
            flow = InstalledAppFlow.from_client_secrets_file(CREDENTIALS_FILE, SCOPES)
            creds = flow.run_local_server(port=0)

        with open(TOKEN_FILE, "w") as token:
            token.write(creds.to_json())
    
    return creds
```

The first time the script is run, a manual login is required. A browser window pops up and you see "You've been given access to an app that's currently being tested. You should only continue if you know the developer that invited you." Clicking Continue authenticates the app, creating a token.json file. This file contains a short-lived (2 hour) access_token to authenticate API requests and a long-lived refresh_token that allows the script to obtain new access tokens in the future without requiring another manual login[^1].

The only thing to do locally is ensure we add credentials.json and token.json to our .gitignore to avoid sharing them with the world. When we deploy to the cloud, we'll need to manually upload our token.json to the cloud server, or SSH into the server and run the script to do the once off manual login process.

### Taking the time to fix my make up

I started out with sending a basic plain text email, using some ChatGPT code as a starting point. It was definitely a case of seeing the code first and understanding and adjusting it second. I learned about the [MIME](https://en.wikipedia.org/wiki/MIME) standard, where the email body has a tree structure, with components including plain text, attachments and alternative plain text / html elements.

I decided it would be nice to attach a data visualisation which shows the historical and forecast rainfall figures the model is basing its recommendation on, when sending through the model's recommendation. This was set up using matplotlib and a bit of back and forth with ChatGPT to get things just the way I liked, with my main contribution being bossy and picking out the [colours](https://matplotlib.org/stable/gallery/color/named_colors.html) (although I probably could have outsourced this as well). What a time to be alive!

At this point I knew I had gone on a tangent and really should stop, but I couldn't help myself. I had to have inline images. This involved adjusting the function to use an \<img> tag to indicate where the image should be inserted, then replacing this with the html that would source the image from a MIMEImage object based on its content-id.

The final function is shown below. We use MIMEMultipart("related") to combine the body with the attached image, and MIMEMultipart("alternative") to provide a text/plain version in case the receipient can only handle this and not the text/html version. I retained the original attachment functionality in case I want to have general attachments (e.g. pdf files) at some point in the future.

```python
def send_email(sender_email: str = "alexgouldblog@gmail.com",
               receiver_email: str = "alextgould@gmail.com",
               subject: str = "Test email",
               body: str = "This is a test email",
               attach_path: str = None) -> None:
    """Send an email using Gmail SMTP with OAuth2 authentication."""

    creds = _get_credentials()
    auth_string = base64.b64encode(f"user={sender_email}\x01auth=Bearer {creds.token}\x01\x01".encode()).decode()

    # Create email message (allows inline images)
    msg = MIMEMultipart("related")
    msg["From"] = sender_email
    msg["To"] = receiver_email
    msg["Subject"] = subject

    # Check if the body includes <img> (inline image placeholder)
    has_inline_image = "<img>" in body

    # Prepare HTML version of body
    html_body = body.replace("\n", "<br>")  # Convert newlines to <br> for HTML formatting
    if has_inline_image:
        html_body = html_body.replace("<img>", '<img src="cid:inline_image" style="max-width:600px; height:auto;">')

    html_body = f"""
    <html>
        <body>
            <p>{html_body}</p>
        </body>
    </html>
    """

    # Remove <img> tag from plain text version
    plain_text_body = body.replace("<img>", "")

    # Attach both plain-text and HTML versions
    msg_alt = MIMEMultipart("alternative")
    msg_alt.attach(MIMEText(plain_text_body, "plain"))
    msg_alt.attach(MIMEText(html_body, "html"))
    msg.attach(msg_alt)

    # Attach file if provided
    if attach_path:
        with open(attach_path, "rb") as file:
            file_data = file.read()

        if attach_path.endswith(('.png', '.jpg', '.jpeg')):
            if has_inline_image:
                # Attach inline image
                mimefile = MIMEImage(file_data, name="inline_image")
                mimefile.add_header("Content-ID", "<inline_image>")
                mimefile.add_header("Content-Disposition", "inline", filename="inline_image")
                msg.attach(mimefile)
            else:
                # Attach as normal image
                mimefile = MIMEImage(file_data, name=attach_path)
                msg.attach(mimefile)
        else:
            # Attach as normal file
            mimefile = MIMEApplication(file_data)
            mimefile.add_header('Content-Disposition', 'attachment', filename=attach_path)
            msg.attach(mimefile)

    # Send email
    try:
        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.ehlo()
            server.starttls()
            server.ehlo()
            server.docmd("AUTH", "XOAUTH2 " + auth_string)
            server.sendmail(sender_email, receiver_email, msg.as_string())
        logger.info("Email sent successfully!")
    except Exception as e:
        logger.error(f"Error sending email: {e}")
```

The image below shows the resulting emails. The first one is using the plain text approach with a simple attachment, while the second one uses html with inline images. I think it was definitely worth the diversion to use html and include the inline image - it's more aesthetically pleasing and accessible to not have to open an attachment.

![]({{site.baseurl}}/assets/img/happy-plants/email_inline.png)

With everything set up, it's time to automate.

## Autobots, Roll Out!

To have the process run on a daily basis, I created a script that would call the various functions, and then used the Windows Task Manager to run the script automatically each day. As with most of the things I did for the first time in this project, it sounded really simple at first...

### Including python modules from another project directory

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

The goal is for daily_run.py in the scripts folder to call the necessary functions from the files in the src folder. Unfortunately, this won't happen by itself. When you `import my_module` in Python, it looks for the module in sys.path, a list of directories that Python searches when importing modules, which by default includes the script's directory, standard library paths, installed site-packages and anything manually added to PYTHONPATH or sys.path. PYTHONPATH is an environment variable used for adding additional directories to sys.path, although modifying it can be cumbersome and varies by OS and execution context. Without having the src directory in either sys.path or PYTHONPATH, daily_run.py can't import modules from the src files.

ChatGPT threw me a red herring, suggesting that I needed to add an \_\_init\_\_.py file in the src folder. This is only relevant if you're distributing your code as an installable package. If you go down this rabbit hole you'll learn all about the changes that were introduced in Python 3.3 (circa 2012), where the \_\_init\_\_.py file ceased to be required by namespace packages - where directories with the same name in different locations are combined into a package - but continued to be required for regular packages. One big fat red herring.

If I were less stubborn, I probably would have moved daily_run.py into src and moved on with life. But I've encountered this issue in the past and decided to come up with a good solution. Well here it is:

```python
import sys
src_path = os.path.abspath(os.path.join(os.path.dirname(__file__), '../src'))
if src_path not in sys.path:
    sys.path.append(src_path)
```

This code is used at the top of daily_run.py to add the src directory to sys.path at run time. It ensures that the correct path is determined regardless of the OS (Windows/Linux) or current working directory. It also avoids the need to take manual steps, such as setting PYTHONPATH in Task Scheduler or adjusting .vscode/settings.json (both of which I did get working, but wouldn't recommend).

With this in place, we can now import as normal, with all the normal intellisense etc:

```python
from get_data import forecast_data, historical_data
import database, prepare_data, pred_models, send_email, create_plots
```

Now that we have a script ready to run each day, we need to tell Windows to run it on a schedule.

### Automation using Windows Task Scheduler

Getting Task Scheduler to run the program each day is relatively straightforward, once you know what to do. You can open Task Scheduler by searching for it in the search bar, then Create Task with the following:

* General: Give it a name (e.g. gather_rainfall_data)
* Trigger: Daily (e.g. 7:30am every day)
* Actions: Start a program
  * Program/script: The pythonw.exe file in your (virtual) environment (e.g. C:\.venv\happy_plants\Scripts\python.exe)
  * Add arguments: The full path to the script file you want to run, using parentheses if there's any spaces (e.g. "D:\Projects\happy-plants\scripts\daily_run.py")
  * Start in (optional): The path to your project root; anecdotally this cannot include quotes, so ideally use file paths with no spaces (e.g. D:\Projects\happy-plants)
* Conditions: (optional) I ticked the Network condition as I need internet access to gather rainfall data
* Settings: (optional): I adjusted these to have the task run after a scheduled start is missed (as my computer isn't necessarily on at 7:30am every day), with some restart options (3 retries with 2 hour delays, in case my internet cuts out)

Note we use pythonw.exe rather than python.exe here so that the process runs in the background, without popping up a terminal window.

And with that, we have an automated process, which collects data from a website, adds it to a database, transforms the data to make a recommendation on manual watering, and sends this to the user along with a pretty data visualisation.

## Me want cookie! Om nom nom nom

When you watch a movie, do you sit through the credits in case there's a funny little extra scene afterwards? Well it turns out these are known as a [post-credits scene](https://en.wikipedia.org/wiki/Post-credits_scene), stinger, end tag, or credit cookie.

If you've made it this far, you'll surely enjoy these bonus topics that cropped up along the way. 

### Logging

Many people who write python code will be familiar with the [logging](https://docs.python.org/3/library/logging.html) module. Rather than using print() statements to write outputs to the terminal, you use logger to classify the same information (e.g. DEBUG, INFO, WARNING), so you (or others) can quickly toggle messages on or off depending on where you are in the development stage. 

The code below gives an insight into the logging process I used, although this example is more complex than you would need in general:

```python
import logging
import os
log_file = os.path.join(os.path.dirname(__file__), f"{os.path.splitext(os.path.basename(__file__))[0]}.log") # e.g. collect_data.py -> collect_data.log
logging.basicConfig(filename=log_file, level=logging.INFO, format="%(asctime)s %(levelname)s: %(message)s", datefmt="%Y-%m-%d %H:%M:%S")
logger = logging.getLogger(__name__)
logger.debug(f"log_file being written to {log_file}")
logging.getLogger('matplotlib').setLevel(logging.WARNING)
logging.getLogger('PIL').setLevel(logging.WARNING)
```

There's a few interesting elements in this:

* **log_file** - similar to the sys.path approach, I define my log_file path in a manner that will work regaredless of the environment in which the file is run. This also means there's no need to manually redirect the log output when running Task Scheduler (I got this working at one point, but wouldn't recommend it). As it stands, the code above creates a log file in the same directory as the script, with the same file name and a .log extension. Omitting this line results in log outputs being written to the terminal (more common and useful during the development stage).
* **basicConfig level** - I set the level to logging.INFO as the code is fully developed, but during development I set it to logging.DEBUG. An interesting developer choice is the extent to which you retain your DEBUG messages once they've served their purpose, versus deleting them, commenting them out or having some sort of `if False:` condition that would allow you to quickly bring them back if needed. My approach to this was to retain DEBUG messages that were generally useful (e.g. a print of the new data being sourced) while deleting ones that were more specific (e.g. breaking apart a line of code that ChatGPT generated so I could understand it), to avoid the code becoming cluttered.
* **basicConfig datefmt** - The default format lacks a time value, which can be quite useful - particularly when you don't yet trust your Task Scheduler and want to confirm it ran correctly in the background the first few times!
* **logger = logging.getLogger(__name__)** - In my src/*.py files I tend to only use this line, as it picks up the logger established using the basicConfig set up in scripts/daily_run.py.
* **logging.getLogger('matplotlib')** - When creating my simple plots to include in the email, I observed a lot of messages cluttering my log. This line shows how you can disable logging from other modules, in this case setting the level to WARNING so as to suppress the INFO spam from the matplotlib and PIL modules.

### Snippets

Another feature that vscode users may already be aware of are snippets. You can access these using Command Palette > Snippets > Configure snippets.

Here are a couple of examples:

```json
{
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

Once they're defined, you can type the "prefix" value and press Tab to have it replaced with the "body" value. This is particularly useful for two reasons:

1. Code which is used frequently (e.g. creating a logger.debug statement to print out the value of some variable) can be added by mindlessly typing a few letters (e.g. log). The cursor can even be placed in the correct location for you to start typing the next bit (e.g. in the example shown above, $1 is where the cursor will move to when Tab is pressed)

2. Standard code which is used infrequently. In the examples above, I have one for my PROJECT_ROOT approach that lets me use paths relative to my script location in a system agnostic manner, and another that adds the standard `python if __name__ == \"__main__\":"` bit at the bottom of my script for testing the script during the development stage. Traditionally, I would find another example of this code and copy and paste it across, but taking the time to create a snippet makes this process faster going forward.

### Docstrings

Similar to using logging rather than print statements, I'm on a journey to improve my use of [docstrings](https://peps.python.org/pep-0257/), with a preference for the [Google docstring style](https://google.github.io/styleguide/pyguide.html#383-functions-and-methods). This is another area where programming can be more art than science.

In addition to getting into the habit of writing and maintaining these myself as I go along, I've found it useful to plug functions into ChatGPT and simply ask for it to add a docstring in the Google docstring style, which generally gets me something decent. This approach is also useful for adding [type hints](https://docs.python.org/3/library/typing.html).

Taking a moment to add docstrings can pay dividends down the line, when you're referring to your own functions from another script and can hover over them (in vscode at least) for an IntelliSense summary that covers the main arguments and/or examples of how to use the function. I'm sure it's appreciated even more by others who didn't write the functions but need to understand them - I certainly find myself benefiting when others add docstrings (and cursing when they don't).

## Conclusion

In this post, we created the Happy Plants system, which now delivers me a daily email containing a data visualisation of recent and forecast rainfall, along with a recommendation of whether I should manually water my plants.

In subsequent posts, we can extend this system in various ways, such as:

* Deploy the system to a cloud platform (e.g. GCP) using a Linux environment with automation using cron/systemd. This could also involve using Terraform to create a reproducable infrastructure model that others can use to deploy the system.

* Explore alternative models for making the watering recommendation, such as reinforcement learning and/or machine learning models. This could also involve creating synthetic data to train the models on (noting that there isn't a history of forecasts, only of actual rainfall).

* Create an API to register when the user waters their plants, which can be accessed by clicking a button in the daily email. This way the system doesn't have to assume that the user has watered their plants, and the user can ignore the recommendation and get a reminder the next day.

That's all for now. I hope you've enjoyed the insights from this project as much as I enjoyed building it!

---
<small>Footnotes:</small>

[^1]: Some time after completing this project I stopped receiving emails as my token was expired. It turns out there's a limit of 50 refresh tokens before a manual login is required again. The code shown handles this, prompting the user to reauthenticate. This works fine on a local system, but for a cloud platform deployment we'll need to revisit our email approach, probably setting up our own domain and/or trying out SMTP Relay.