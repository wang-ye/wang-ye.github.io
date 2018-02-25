Recently I worked on a project involving lots of datetime conversion and comparisons. This is error-prone, so I created a cheetsheet to cover the common datetime tasks.

## General Principles

### Do Not Use Time Package for Timezone Transformations
It is very error-prone. Use Datetime and pytz packages instead.

### Timezone Awareness
Python’s datetime.datetime objects have a tzinfo attribute. It stores time zone information with a `datetime.tzinfo` subclass instance. When this attribute is set, a datetime object is called `aware`. Otherwise, it’s `naive`.

`datetime.datetime.now()`, without providing the tz attribute, is a native datetime. It shows the current local date and time. If your servers are running in multiple timezones, the datetime.now() can cause confusions.

If timezone conversion is needed, making all datetime *aware* is a good option. This makes the transformation simpler.

## Code Snippets
Here are some common code snippets.

### Get Current Datetime

```python
# Local time without timezone info.
datetime.datetime.now()
# Current time in UTC representation, but still without tzinfo
datetime.datetime.utcnow()
# Current time in UTC representation, with tzinfo
datetime.datetime.utcnow().replace(tzinfo=pytz.utc)
```

### Get Local Timezone
The `tzlocal` package can do the trick.

```python
from tzlocal import get_localzone
local_tz = get_localzone()
```

### Get Epoch (Timestamp)

```python
def utc_timestamp(utc_dt):
    return (utc_dt - datetime.datetime(1970, 1, 1)).total_seconds()

print(utc_timestamp(datetime.datetime.now()))
```

### Get Date of a Datetime
This is simple, just call `date`!

```python
datetime.now().date()
```

### Convert Between Different Timezones
```python
# Convert utc_ts to pdt_ts
def to_pdt_time(utc_ts):
    from_zone = pytz.utc
    to_zone = pytz.timezone('US/Pacific')
    pdt_ts = utc_ts.replace(tzinfo=from_zone).astimezone(to_zone)
    return pdt_ts
```

### String To Aware Datetime
We use `strptime` for this conversion.

```python
>>> from datetime import datetime
>>> datetime.strptime('2009/05/13 19:19:30 -0400', '%Y/%m/%d %H:%M:%S %z')
datetime.datetime(2009, 5, 13, 19, 19, 30,
                  tzinfo=datetime.timezone(datetime.timedelta(-1, 72000)))
```

### Datetime To String
We use `strftime` for this conversion.

```python
from datetime import datetime
current_time = datetime.now()
print(current_time.strftime("%m/%d %H:%M:%S"))
```

## Summary
This post covers some common datetime/timezone conversion snippets. Datetime handling can be confusing and error-prone, so use the right approach!