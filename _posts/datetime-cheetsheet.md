Recently I worked on a project involving lots of datetime conversion and comparisons. Here is a cheetsheet for some of the common datetime tasks.

## General Concepts

### Do Not Use Time for Timezone Transformations
It is very error-prone. Use Datetime and pytz packages instead.

### Timezone Awareness
Python’s datetime.datetime objects have a tzinfo attribute. It stores time zone information, in the form of datetime.tzinfo subclass instance. When this attribute is set and describes an offset, a datetime object is aware. Otherwise, it’s naive.

datetime.datetime.now(), without providing the tz attribute, is a native datetime. It shows the current local date and time. If your servers are running in multiple timezones, the datetime.now() can cause confusions.

If timezone conversion is needed, making all datetime *aware* is a good option. This makes the transformation simpler.

## Code Snippets
### Get Current Datetime

```python
# Local time without timezone info.
datetime.datetime.now()
# Current time in UTC representation, but still without tzinfo
datetime.datetime.utcnow()
datetime.datetime.utcnow().replace(tzinfo=pytz.utc)
```

### Get Local Timezone
datetime + pytz

### Get Epoch (Timestamp)

```python
def utc_timestamp(utc_dt):
    return (utc_dt - datetime.datetime(1970, 1, 1)).total_seconds()

print(utc_timestamp(datetime.datetime.now()))
```

### Get Date of a Datetime

Datetime

### Parse a Datetime string
2018-01-22T11:34:16.84196Z
utc = datetime.datetime.strptime(ts, '%Y-%m-%dT%H:%M:%S.%fZ')

```python
>>> from datetime import datetime
>>> datetime.strptime('2009/05/13 19:19:30 -0400', '%Y/%m/%d %H:%M:%S %z')
datetime.datetime(2009, 5, 13, 19, 19, 30,
                  tzinfo=datetime.timezone(datetime.timedelta(-1, 72000)))
```

### Convert Between Different Timezones
```python
def to_pdt_time(ts):
    """Only return side, amount, price and ts."""
    from_zone = pytz.utc
    to_zone = pytz.timezone('US/Pacific')
    utc = datetime.datetime.strptime(ts, '%Y-%m-%dT%H:%M:%S.%fZ')
    pdt = utc.replace(tzinfo=from_zone).astimezone(to_zone)
    return pdt
```

### Output Timezone As String
return pdt.strftime("%m/%d %H:%M:%S")

## Summary
