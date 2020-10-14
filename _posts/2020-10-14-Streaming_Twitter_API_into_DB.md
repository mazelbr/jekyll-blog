---
layout: post
title:  "Connecting Python and MySQL"
tag: python twitter api tweepy
---


# How to Stream Twitter Data into DB's

In this Tutorial I want to use the `tweepy` package to access the twitter API and get a live stream of incoming tweets directly into a database

## An overview
`tweepy` is a wrapper for the official twitter API. The API does provide a filtered stream API where one can customize filters (e.g. contains a word, has hashtags, region of origin ...).

Tweepy provides a `Stream` class that can hook up onto a stream and in which one can define filters. To actually get tweets the `StreamListener` class which takes a `Stream` object as a argument. Inside the `StreamListener`we have `on_data` method that gets called every time a new tweep is posted.

To give a quick overview what the plan is, the data collection part consists of two modules. In one of them we inherit the `StreamListener` and overwrite its `on_data` method so we can define what happens with new tweets. The second part is a script which ideally would run forever on a server. It should create a `Listener` object run it and take care of the error handling.

We further want to set up a database or more precisely test different DBses against each other.


## Notes on the Twitter API
Both, the documentation of tweepy as well as the one of twitter explain that inside the `on_data` method no heavy computations should take place. The reason is that the Stream does not care about us and simply pushes new tweets into our application. If our application is still busy with anything else and can not listen then the new tweet is stored in a buffer. However if we constantly are slower than the stream, at some point the buffer will be full and twitter ends the Stream. In such case a `IncompleteRead` error is raised which leads to a `url3lib.ProtocolError` in the running script. 
>It is worth noticing that delaying is not the only can cause `IncompleteRead`s. Sometimes the api sends incomplete data which can not be prevented at all.
In a short case study catching all tweets in England, the buffer was full after 13 Minutes if a delay of 10 seconds was introduced in the on_read function.

In theory `tweepy` has built in helper methods that should inform about `stall_warnings`(when the buffer tends to get to its limit) and acts on in with a `on_warning` methods. However, apparenly that functionality is currently not working.

In addition the Number of Tweets during a day and is limited to 2400 tweets. It has further limitations for each hour or half an hour.

## Implementation
The first step is to connect and authenticate to the API. We should have created an Application on the Twitter developer dashboard and store our credentials somewhere safe. Then we can use the `tweepy.API` method. We wrapped that up in a own method for better readability.

```python
def authenticate(creds = "credentials.txt"):
    
    with open(creds) as creds:
        c_key, c_secret, a_token, a_secret = creds.read().split()
    
    auth = tweepy.OAuthHandler(c_key, c_secret)
    auth.set_access_token(a_token, a_secret)
    api = (tweepy.API(auth,wait_on_rate_limit=True, #Streaming: API waits if rate limit is reached and gives out an notification 
    wait_on_rate_limit_notify=True))
    return api
```

As mentioned we will inherit from the tweepy `StreamListener` class to modify the data handling.
```python
class WriteFileListener(StreamListener):
    
    def __init__(self, max = 10000):
        self.stop = False
        self.max = max
        self.c= 0
        super(StreamListener, self).__init__()
```
We also added a counter and a `max` attribute after which the streaming should stop. The `stop` attribute will be used to end the stream.

The most important step is the overwriting of the `on_data` method. For this one can refer to the sourcecode of the original class on [Github](https://github.com/tweepy/tweepy/blob/master/tweepy/streaming.py).
The raw data comes in a a string which denotes a python object. We use the `json` modul to get back the original object.

```python
def on_data(self, tweet):
        text= json.loads(tweet).get("text",'EMPTY')
        # DO STUFF
        self.c +=1
        if (self.c >= self.max) or self.stop:
            return False
```

>Noteworthy about the `on_data` method is, that it continues as long it returns `True`, which also means that returning `False` will properly end the Stream. The idea is to 

## The application file
The application file is the one that should run ideally constantly on a server. It sets up the API and the Stream. On errors it should handle them properly e.g. by restarting the Stream or waiting after we reached the limit.

```python
api = authenticate()
listener = WriteFileListener(max=500)
try:
        stream = tweepy.Stream(auth=api.auth, listener=listener, tweet_mode='extended')
        stream.filter(languages=['en'],locations=[-10.85, 49.82,2.02, 59.47], is_async=False,stall_warnings=True)    
    except:
        #Error Handling HERE
        raise
    finally:
        listener.stop = True
        stream.disconnect()
```
This is basically the boilerplate. As the `is_async`attribute indicates, the `Stream` supports asynchonous behaviour. If we set this to true python will continue with the rest of the script and handles new tweets as they come in. In general this might be a good idea, but note that you have also take care of that in the `on_data` method when e.g. writing the Tweets somewhere because parallelized writes can be problematic (solvable though).

The `stall_warining` should in theory allow you to access warning, when we are handling data to slow. But apparently this is currently not working.

## Discussing Design Decisions
Apparently processing speed is a topic to not fall behind the Stream. There might be several ways to solve that.
### Async vs Sync.
From my current understanding async is a simple version of threating. This allows concurrency instead of sequential processing. (Note this is not the same as parallelism). Instead of waiting for a tweet to be fully processed, async allows a more efficient use of waiting times. In case one tweet has to wait of a response (e.g.) from the sql server, another tweet can already be processed. This might speed things up and solve the buffer problem. However python does not allow it to have a shared connection over different threats. This means instead of making connection to a file or db at the beginning of the `try` statement and then passing it over to the stream, which then simply has to write to it, we have to make a connection inside the `on_data` method for every tweet seperately. Especially for text files this might slow down the process since searching the file and opening the connection takes way more time than sequential write.
Since text files and databases use `locks` we introduce waiting times which on the other hand could be used efficiently thanks to asyn.

In the case of databases another variable should be taken into account. Accoriding to [SQLite](https://www.sqlite.org/faq.html#q19)
>Actually, SQLite will easily do 50,000 or more INSERT statements per second on an average desktop computer. But it will only do a few dozen transactions per second. Transaction speed is limited by the rotational speed of your disk drive. A transaction normally requires two complete rotations of the disk platter, which on a 7200RPM disk drive limits you to about 60 transactions per second. 

Having one connection would speed up the writing while due to `ACID` conditions more commits would be safer since on exception we would not loose all the writes but only one.

Which way might be the best is not fully evaluated yet, but a few dozen commits seem alright and we might already be fast enough anyways.

### The choice of the database
Currently all tweets are saved as strings in a text file. This might not be the best choice since those files are not optimized at all. Remeber that a script could throw exceptions at any time and breaking during a write might corrupt the whole text file. For exactly that purpose `ACID` conditions were developed for databases so maybe we should use them. In the following we might discuss and explain the use of `SQLite` as a textfile-like storage, `MySQL` as a traditional solution and consider `Redis` of both are not fast enough.

Currently we perform all the database related tasks inside the `on_data` method. The `SQLite` is by far the easiest one. The core it the `execute()` method, in which we can write almost any SQL Syntaxt. To make it pythonic we can wrap it up in helper methods. As physical storage we can use whatever file we want. If the specified one is not existing `SQLite` creates it for us. 

```
def make_connection(name = "tweet.db"):
    con = sqlite3.connect(name)
    return con

def show_table():
    with make_connection() as conn:
        c = conn.cursor()
        c.execute("SELECT name from sqlite_master WHERE type='table';")

def make_table():
    with make_connection() as conn:
        c = conn.cursor()
        c.execute("CREATE TABLE tweets (id INTEGER PRIMARY KEY, json text)")
```
After creating a table we can start storing tweets into it. In the current implementation we store the text into a `TEXT`type column. Another option might be to store the `json` object as a `BLOB`. Since we are using "user" input from the internet we must use the `?` notation to prevent `SQL-Injections`. The value that we want to store comes as a list as another argument.

```python
text= json.loads(tweet).get("text",'EMPTY')
 with sqlite3.connect("tweet.db", isolation_level=None) as conn:
            db_cursor = conn.cursor()
            db_cursor.execute("INSERT INTO tweets(json) VALUES (?)", [text])
            #print("tweet")
            self.c +=1
```

That is all we need and the SQLite database is now set up.

### MySQL
MySQL definately need more set up until it is running. We will discuss the detailed implementation in an other documentation. A benefit of the MySQL approach could be that it is safe to use in terms of dataloss and a professional reliable tool. Since MySQL runs a server it has a own buffer which might relax the buffer problem at the api. Furthermore the python program then does not need to hanldy any writes. All the data stays constantly in memory and MySQL deals with the writing. This could harmonize quite well with `async` enabled.