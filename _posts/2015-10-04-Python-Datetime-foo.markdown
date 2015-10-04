---
layout: post
title:  "Python Datetime" 
date:   2015-10-04 
author: Abhijit Gadgil
categories: Python
tags: python, datetime
---

Recently, I had to insert some data into an [SQLite3](https://sqlite.org) database. One of the columns was a Date. Since SQLite3 doesn't support a direct date column but supports convenient functions for handling date, one's left with a question, what's a best choice for storing dates in SQLite3 DB. Of course integers are perfect and SQLite supports them. But when you want to store date as integers, specifically something like Unix timestamps, one has to deal with a number of issues, most notably is the timezone. The time offsets could mess things around. I'd had this trouble in the past when I developed code on my Laptop in IST timezone and then deployed it on a server in UK. I had somehow figured around it back then, mostly in a hacky way and got it going and I faced same problem again! (for a very related purpose). I thought it's worthwhile to compile this at one place for my reference.

I'd be using python `time` and `datetime` modules for this. Python's `time` module has got a function `time.time()`, which gives number of seconds since Unix Epoch (1st Jan 1970 00:00 UTC). Note, this is just a float and if the local time is synced on machines - doesn't matter what timezone one is in `time.time()` is always going to return same float.  So if I have to store a date in SQLite3, all I have to do is - really save this number and use various date-time functions to read it as a date. Also dealing with numbers is a lot easier. The catch is the python's functions and the timezones make it a little tricky.

So here's the problem statement - I have a date string say in the format 'YYYY-MM-DD'. I have to store this as a number in a date field in database. And subequently, I want to be able to read this number and get back the same date in the above format. Regardless of which timezone I run my code in. 

So I quickly wrote a script to do this. I have used `time.mktime` to get the right timestamp. This timestamp will be stored in the DB. The trouble with `time.mktime` is, it doesn't accept timezone as an argument but only works on local timezone. Interestingly, one can use `time.tzset` to set a timezone to appropriate value. [This question on SO](http://stackoverflow.com/questions/530519/stdmktime-and-timezone-info) discusses the same issue and proposes an approach - where you set `TZ` environment variable and then call `time.tzset` to use the 'UTC' timezone, then call a `time.mktime` to get the right tuple. This can then be used with something like `datetime.datetime.utcfromtimestamp` again (or SQLite Datetime function with 'unixepoch' modifier). Below's the code that does this.

{% highlight python %}
# A simple script that uses mktime with right environment set for UTC timestamps
import os
import time
from datetime import datetime as dt
import sys

def get_ts_for_datestr(date='2015-10-04', fmt='%Y-%m-%d'):
    if os.environ.has_key('TZ'):
        old_tz = os.environ['TZ']
    else:
        old_tz = None
    os.environ['TZ'] = ''
    time.tzset()
    ts = time.mktime(time.strptime(date,fmt))
    if old_tz is not None:
        os.environ['TZ'] = old_tz
    else:
        os.environ.pop('TZ')
    time.tzset()
    return ts

def get_datestr_from_ts(ts, fmt='%Y-%m-%d'):
    d = dt.utcfromtimestamp(ts)
    return time.strftime(fmt, d.timetuple())

# To run the script, don't forget to pass the date as command like argument 
print get_datestr_from_ts(time.mktime(time.strptime(sys.argv[1],'%Y-%m-%d')), '%Y-%m-%d')
print get_datestr_from_ts(get_ts_for_datestr(sys.argv[1],'%Y-%m-%d'), '%Y-%m-%d')

{% endhighlight %}

Most of the code is self explanatory. In place of the `get_datestr_from_ts`  above, we could directly use SQlite functions as follows 

{% highlight sqlite3 %}
Datetime(1443978000, 'unixepoch') # Does what we've done above
Datetime(1443978000, 'unixepoch', 'localtime') # takes into consideration local time as well
{% endhighlight %}

Hopefully, I don't need to 'understand' this again and can just refer to this post. 
