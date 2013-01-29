Decoding JSON is bulk in Python
===============================

Lately I've been creating a fair number of APIs that store some JSON in a field
in a database. Often this is a good reason to use a document database like Mongo
or Couch, but sometimes a relational DB will do you better (e.g., if it's just a
small amount of data for each row that is free-form, but the relational aspects
are many and important). If you are using a relational DB with a JSON field, you
will at some point have to deserialize that JSON for use as data.

I'm concerned with the speed of my APIs, so I began wondering what the best way
to do this was. When returning a single record, there aren't many choices. I
load the record, pull out the JSON field value as a string, deserialize it, and
put it back on as a dictionary. However, when returning a potentially large set
of records, my first inclination was to process each record individually, and
compose them all into a list. Something like the following:

        def get_record(...):
            record = execute_query(...)
            data = process_record(record)
            return data

        def get_many_records(...):
            records = execute_query(...)
            data = [process_record(record) for record in records]
            return data

        def process_record(record):
            json_str = record['blob']
            record['blob'] = json.loads(json_str)
            return record

The issue here is that, if I have a number of records, `json.loads` will get
called once for each one. The other option that I had was to compose all of the
JSON data into a single list, deserialize it all at once, and then partition it
back out to its original objects -- something like:

        def get_many_records(...):
            records = execute_query(...)
            json_str = '[%s]' % ', '.join(record['blob'] for record in records)
            blobs = json.loads(json_str)
            for record, blob in zip(records, blobs):  # izip for Python 2.x
                record['blob'] = blob
            return records

My first thoughts: I prefer the first code block, because it allows me to share
code between my single- and multi-record getters, and it seems clearer. Also, I
don't really know yet whether the latter would gain me anything significant. To
test, I wrote up the following:

        from timeit import Timer

        list_str_code = """
            import json
            list_str = '[' + ', '.join(['{"a": 1}']*1000) + ']'
            data = json.loads(list_str)
        """

        str_list_code = """
            import json
            str_list = ['{"a": 1}']*1000
            data = [json.loads(dict_str) for dict_str in str_list]
        """

        list_str_timer = Timer(list_str_code)
        str_list_timer = Timer(str_list_code)

        print list_str_timer.repeat(number=1000)
        print str_list_timer.repeat(number=1000)

On my machine with Python 2.7, the `list_str_code` ran consistently more than 3
times faster than `str_list_code` (1.2 vs 3.9 seconds). With Python 3.2 it was
nearly 5 times faster (0.7 vs 3.3 seconds). That's pretty significant.

It is worth noting that I tried this with lists of different sizes as well. Even
if I construct `list_str` and `str_list` each with only 10 elements and run the
code 100,000 times, the `list_str_code` is still several times faster.
