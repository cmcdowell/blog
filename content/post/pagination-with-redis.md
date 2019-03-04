---
title: "Pagination With Redis"
date: 2019-03-03T10:31:42Z
draft: false
---
![Redis Logo](/redis-logo.png)

Pagination is almost universal in web applications and useful if you want to
deliver manageable chunks of a large data set to the end user at their own
pace. Although pagination can be achieved at the application level it is most
often a task offloaded to the database. This article will explore different
methods for pagination using Redis.

This article will use the example of paging a collection of blog posts for a
blog.

### SSCAN ###

The [SSCAN](https://redis.io/commands/sscan) command can be used for incremental iteration over your collection
of blog posts. The command will return one increment *(page)* of results and
also return a cursor allowing you to get the next increment. When the returned
cursor is `0` then the end has been reached.

Let's look at an example. First, let's assume we have 10 blog posts stored in
Redis using the [SET](https://redis.io/commands/set) command.

    redis> SET post:1 "Super interesting blog post about something."
    "OK"

Now let's use the Redis set data structure as an index of posts.
We can add the key of each post, to this *posts* set.

    redis> SADD posts post:1 post:2 post:3 post:4 post:5 post:6 post:7 post:8 post:9 post:10

When a user wants a page of blog posts we can use SSCAN to retrieve a page of
keys. Supposing our page size is 2, we can do that as follows.

    redis> SSCAN posts 0 COUNT 2
    1) "10"
    2) 1) "post:1"
       2) "post:3"

The above command scans the set of posts with an initial cursor of `0`. Redis
returns the cursor, which can be used with SSCAN to get the next increment of
posts, and the keys of two posts from our posts set.  With each increment, we
can use the [MGET](https://redis.io/commands/mget) command to get the content for our page of blog posts.

    redis> MGET post:1 post:2
    1) "Super interesting blog post about something."
    2) "Yet another great blog post."

When the returned cursor
is `0` you have iterated over the entire set of posts, and you should stop.

You will notice that the posts are returned in a random order, if the order is
important you can use a sorted set as your index of posts and do the same as
above with the [ZADD](https://redis.io/commands/zadd) and
[ZSCAN](https://redis.io/commands/ZSCAN) commands.

As noted in the Redis documentation, SSCAN will not always return **COUNT** items
from your set. Sometimes it can return 0 items with a non zero cursor (remember your iteration will not be
finished until the cursor is 0) or more than **COUNT** items. If the exact number
of items per page is important for your needs you may have to take this
into account.

SSCAN stores no state, so you will either need the user to pass in their cursor
with each request for a page or store it in their session.


### ZRANGE ###

In a similar way to the previous method, we can use one of Redis's compound data
types as an index of blog posts. Just like before we let's assume we have 10
blog posts stored using the [SET](https://redis.io/commands/set) command.

    redis> SET post:1 "Super interesting blog post about something."
    "OK"

This time lets use a sorted set as our index. Just like before we can use the [ZADD](https://redis.io/commands/zadd) command
to add each post to the index as we create it, but this time ZADD lets us give
each element a score. If we use the timestamp of each post as the score this
means our index will be sorted by date/time, which seems desirable for a blog.

    redis> ZADD posts 1551616976 post:1
    (integer) 1
    redis> ZADD posts 1551617014 post:2
    (integer) 1
    redis> ZADD posts 1551617037 post:3
    (integer) 1
    redis> ZADD posts 1551617064 post:4
    (integer) 1

Now assuming a page size of 2, we can retrieve a page with the [ZRANGE](https://redis.io/commands/zrange) command.

    redis> ZRANGE posts 0 1
    1) "post:1"
    2) "post:2"

And for page 2.

    redis> ZRANGE posts 2 4
    1) "post:3"
    2) "post:4"

Note that the start and end index of the ZRANGE command are both inclusive. If you would
prefer the posts in reverse order (newest first) you can use the [ZREVRANGE](https://redis.io/commands/zrevrange) command.

And just like before we can get the content of the posts with the [MEGET](https://redis.io/commands/mget)
command.

The user will have to pass in the start and end index with each request. Or if
you prefer you can convert a page number and a page size to a start and end
index.


## Conclusion ##

There are various methods for pagination using Redis, above are just a few.
Redis is a very flexible database that you can build some fairly complex
features with, and because all the data is stored in memory Redis is very fast.
