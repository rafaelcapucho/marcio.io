+++
date = "2016-09-05"
title = "Speeding up your MongoDB queries up to 30 times with Tornado"
type = "post"
ogtype = "article"
+++


Sometime ago, I was facing a *problem* using MongoDB to perform heavy operations like Aggregations over a considerable amount of documents.

Fortunately, most of them were repeating, like when your projects are fetching contents from database to make the same menu structure and don't change every second.

We already know how is the recipe to solve that: **Cache it**.

I was using [Tornado](http://www.tornadoweb.org) and it's common to use [Motor](https://github.com/mongodb/motor) to perform async database operations over MongoDB. The idea was to implement a Mixin that can be used in all handlers and support the three most common operations to get data:

1. Find
2. Find One
3. Aggregate

Ok, let's do it.

## Establishing the connection

We won't need a lot of stuffs here, just the boilerplate.

```python
import logging
import motor

from pymongo.errors import ConnectionFailure
from tornado.options import define, options

define("mongo_host", default="mongodb://localhost:27017", help="mongodb://user:pass@localhost:27017")
define("mongo_db", default="database_name")

def create_connection():
    try:
        return motor.MotorClient(options.mongo_host)[options.mongo_db]

    except ConnectionFailure:
        logging.error('Could not connect to Mongo DB. Exit')
        exit(1)
        return False
```

Now we're able to establish the connection and the object will be available through `self.application.db` in every handler. Other aspects of the code like imports are omitted.


```python
class Application(tornado.web.Application):
    def __init__(self):
        handlers = [
            (r"/?", MainHandler),
        ]

        self.db = create_connection()

        tornado.web.Application.__init__(self, handlers)

http_server = tornado.httpserver.HTTPServer(Application())
http_server.bind(options.port)
http_server.start(1)

main_loop = tornado.ioloop.IOLoop.instance()
main_loop.start()
```

## The Mixin Code

With the connection available, it's time to our Mixin.


```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import print_function
from __future__ import unicode_literals

from hashlib import md5
import datetime

import tornado.web
from tornado import gen
from tornado.options import define, options

from handlers import DBMixin

define("cache", default="True", type=bool, help="Enable/Disable the internal cache")

_cache = {}

class DBMixin(object):
    @property
    def db(self): return self.application.db

class CacheHandler(tornado.web.RequestHandler, DBMixin):
    def __init__(self, *args, **kwargs):
        super(CacheHandler, self).__init__(*args, **kwargs)
        self.FIND_ONE = 1
        self.FIND = 2
        self.AGGREGATE = 3

    @gen.coroutine
    def cache(self, type, col, *args, **kwargs):

        memory = kwargs.pop('memory', True) # Local Memory (Ram)
        timeout = kwargs.pop('timeout', 60)
        sort = kwargs.pop('sort', None)

        # Union of every input
        signature = str(type)+col+str(args)+str(kwargs)

        # Generate a unique key 
        key = md5(signature.encode('utf-8')).hexdigest()

        @gen.coroutine
        def get_key(key):

            if not options.cache:
                raise gen.Return(False)

            if memory:
                if _cache.get(key, False):
                    raise gen.Return(_cache[key])
                else:
                    raise gen.Return(False)

            else:
                data = yield self.db['_cache'].find_one({'key': key})
                raise gen.Return(data)

        @gen.coroutine
        def set_key(key, value):

            delta = datetime.datetime.now() + datetime.timedelta(seconds=timeout)

            if memory:
                _cache[key] = {
                    'd': value,
                    't': delta
                }

            else:
                yield self.db['_cache'].insert({
                    'key': key,
                    'd': value,
                    't': delta
                })

        @gen.coroutine
        def del_key(key):
            if memory:
                if _cache.get(key, False): del _cache[key]
            else:
                yield self.db['_cache'].remove({'key': key})

        _key = yield get_key(key)

        if _key:

            # If the time in the future is still bigger than now
            if _key['t'] >= datetime.datetime.now():
                raise gen.Return(_key['d'])
            else: # Invalid
                yield del_key(key)

        # otherwise (key not exist)
        if type == self.FIND_ONE:

            data = yield self.db[col].find_one(*args, **kwargs)

        elif type == self.FIND:

            if sort:
                cursor = self.db[col].find(*args, **kwargs).sort(sort)
            else:
                cursor = self.db[col].find(*args, **kwargs)

            data = yield cursor.to_list(kwargs.pop('to_list', None))

        elif type == self.AGGREGATE:

            cursor = self.db[col].aggregate(
                kwargs.pop('pipeline', []),
                *args,
                cursor = kwargs.pop('cursor', {}),
                **kwargs
            )

            data = yield cursor.to_list(kwargs.pop('to_list', None))

        if options.cache:
            # Persist the key
            yield set_key(key, data)

        raise gen.Return(data)
```

## How it works?

The code is simple, basically we can use Local Memory or a MongoDB table as cache, and we are able to choose which one by setting the argument `memory`.

Also, your handler should be using our Mixin, as example.

```python
class MainHandler(tornado.web.RequestHandler, CacheHandler):

    @gen.coroutine
    def get(self):

        category = yield find_category_by_slug('electronic-gadgets')

        self.render('index.html', {'category': category})

    @gen.coroutine
    def find_category_by_slug(self, slug, filters={}):

        defaults = {'slug': slug}
        defaults.update(filters)

        category = yield self.cache(
            self.FIND_ONE, # Type of query
            'categories',  # Name of the document
            defaults,      # Conditions
            memory=True,   # Store on python memory 
            timeout=3600   # Expire in 1 hour
        )

        raise gen.Return(category)

    @gen.coroutine
    def find_categories(self, slug, filters={}):

        categories = yield self.cache(
            self.FIND,              # Type of query
            'categories',           # Name of the document
            filters,                # Conditions
            sort = [('order', 1)],  # Sort Criteria
            memory = False,         # Store on db table
            timeout = 86400,        # Expire in 1 day
            to_list=50              # Return up to 50 items
        )

        raise gen.Return(categories)

    @gen.coroutine
    def complex_aggregation(self, category, sort, limit):

        pipeline = [
            {'$match':
                {'cats_ids': {'$all': [ ObjectId(category['_id']) ]}}
            },
            {'$project': 
                {'products': 1, 'min_price': 1, 'max_price': 1, 'n_products': 1, '_id': 1}
            },
            {'$sort': sort },
            {'$skip': self.calculate_page_skip(limit=limit)},
            {'$limit': limit}
        ]

        groups = yield self.cache(
            self.AGGREGATE,      # Type of query
            'products_groups',   # Name of the document
            memory=False,        # Store on db table
            pipeline=pipeline    # Operations
        )

        raise gen.Return(groups)
```

As you can see, there are always an opportunity to enhance the performance.


[Download the Gist](https://gist.github.com/rafaelcapucho/571a5ce63bc0fb0d4c43b445324ae58c)


### How can I know if it worth the price?

It doesn't count if you don't see it, right? Ok, lets measure it.


```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

from __future__ import print_function
from __future__ import unicode_literals

import time

class Timer(object):
    def __init__(self, verbose=False, desc=""):
        self.verbose = verbose
        self.desc = desc

    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, *args):
        self.end = time.time()
        self.secs = self.end - self.start
        self.msecs = self.secs * 1000  # millisecs
        if self.verbose:
            if self.desc:
                print('Time elapsed of {0} - '.format(self.desc), end="")
            print('{0:f} ms'.format(self.msecs))
```

Now we are able to test it.

```python
with Timer(True, desc='getting the category') as t:
    category = yield self.cache(self.FIND_ONE, 'categories', {'slug':'cars'}, memory=True)

with Timer(True, desc='getting the category') as t:
    category = yield self.db.categories.find_one({'slug': 'cars'})
```

I'm pretty sure that you will see the spent time near to 0ms with the cache.

Cheers,  
Rafael.
