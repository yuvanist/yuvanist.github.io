# The One With Better Django Queries



Okay, It has to be said first. Django is one of the prettiest frameworks you could work with. It offers everything under the hood, but as Uncle Ben used to say,

> With great power there must also come great responsibility

Not using things properly or the right way could cause a lot of issues. So let's go ahead and see some Django queries and how it performs under different conditions.

All the Django Queries discussed below are applied on the **Benchmark** table which has no Index. Here is the structure of it.

```py
class Benchmark(models.Model):
    knowledge_begin_date = models.DateTimeField(null=False)
    knowledge_end_date = models.DateTimeField(null=True)
    client_id = models.IntegerField(null=False)
    databook_id = models.UUIDField(null=False)
    datasheet_id = models.UUIDField(null=False)
    data = JSONField(null=True, encoder=DjangoJSONEncoder)
```


In case, if you wanted to play around the table, you can clone the Django App [here](https://github.com/yuvanist/django-app-to-test-queries)

I have inserted 2 Million records into the DB table before firing the queries. You can download the database dump [here](https://drive.google.com/drive/folders/1AgWlTMV5Vg5s9OglVHa7AtXDau1s3FPI?usp=sharing). To insert more rows you can use the script from the git repo mentioned above.

## 1. Difference between only, defer, values and values_list

Let's understand these four things with a single objective query. We have to get all the Unique databook_id from our table. That's it. That's all we have to do.
    
#### Using .only

Consider a fat model with tons of columns, fetching an entire object will be a huge overhead instead of fetching a subset of columns.

```py
benchmark_objects = Benchmark.objects.only('databook_id')

unique_databook_ids = set()
for each_object in benchmark_objects:
    unique_databook_ids.add(each_object.databook_id)

try:
    print(benchmark_objects[0].data)
except:
    print('Not able to access data') 
```

We took all the objects but only with the *"databook_id"* column. This is far better than fetching an entire object and iterating over it for *"databook_id"*.

Now let's look closely at the *"try-and-except"* block. Here I was trying to access the column *"data"* which I didn't add in .only() function as paramter.

If you have thought Django would throw an Exception for doing this, don't worry. you are not alone. But interestingly what happens is One more query will be fired and the data column will be fetched from the table, so there won't be any Exceptions. weird, right?

Now coming to the time part, It took **23.72 Seconds** in my machine to run this block. Costly. I know.


### Using .defer()

Exact opposite of .only(). It fetches everything apart from what is specified in .defer()

```py
benchmark_objects = Benchmark.objects.defer('data')

unique_databook_ids = set()
for each_object in benchmark_objects:
    unique_databook_ids.add(each_object.databook_id)
    
try:
    print(benchmark_objects[0].data)
except:
    print('Not able to access data')

```

Nothing fancy here, same as above, take the object without *data* column which is a JSON field and iterate over it for *databook_id*.
Again, look closely at the *"try-and-except"* block statement, if you had thought at least here Django would throw Exception.

<div class="tenor-gif-embed" data-postid="18429137" data-share-method="host" data-aspect-ratio="1.5" data-width="50%"><a href="https://tenor.com/view/sorry-arnab-i-am-sorry-i-am-sorry-babu-gif-18429137">Sorry Arnab GIF</a>from <a href="https://tenor.com/search/sorry-gifs">Sorry GIFs</a></div> <script type="text/javascript" async src="https://tenor.com/embed.js"></script>

Django will fire one more query and fetch *data* column.
This block took whopping **41 Seconds** to execute. which is understandable, considering the fact it is fetching all other columns apart from *data*.

### using .values()

If you check .only() and .defer(), you can see that the return type is Object. That's why we were accessing it as *object.column_name*. .values() will return a list of dictionaries instead of a model object. It also allows us to select only the set of columns we need to fetch from the model. 

```py
benchmark_objects = list(Benchmark.objects.values('databook_id'))

unique_databook_ids = set()
for each_object in benchmark_objects:
    unique_databook_ids.add(each_object['databook_id'])
    
try:
    print(benchmark_objects[0]['data'])
except:
    print('Exception: Data is not accessible here')
```

A list of dictionaries will be stored in *benchmark_objects* and we are iterating over the list to fetch *databook_id*
This block took **10.5 Seconds** to execute. which is much better than the previous two.

### using .values_list(flat=True)

Cool method. just returns the "databook_id" as a flat list. Removing the *flat=True* would give a list of tuples.

```py
unique_databook_ids = set(
    Benchmark.objects.values_list("databook_id", flat=True)
)
```

It got executed in **6.72 Seconds**. 

Okay, Okay, I hear you. This is also slow. But did you see something, which is common in all the four blocks above?

We took all the objects from the table and added them to a *set*. What if the database does that thing instead of Python, what would be the performance improvement then? 

```py
unique_databook_ids = set(
    Benchmark.objects.values_list("databook_id", flat=True).distinct()
)
```

Just added **.distinct()** to the query. So the table does the unique filtering. Interestingly, it took only **0.8 Seconds** to execute the block.

If you have read it till now, Thank you. You have a nice attention span!

Coming back, Remember how we were doing this same operation in 40 seconds or 30 seconds. Now we did the same thing in under 1 second. 

> Things to Note:
> 1. Take only what you need from the table. Don't be greedy.
> 2. Do it in the table if you can, if there is no way then use The Python vanilla functions.

## 2. Don't bring a Knife to a Gunfight.

Suppose you have to write a block of code that executes only when there is a particular entry available in the table, we can do this in multiple ways. Like,
1. Filtering out all the entries with our condition and applying **len()** function on it.
2. Filter and do **.count()** on the query, which returns the number of entries which match our condition.
   
As a wise person, you followed the second option remembering what we discussed before ie, **Do it in the database if you can**.

```py
db_ds_objects = Benchmark.objects.filter(
    databook_id="61722a62-fe71-44df-86a5-477bcdfbd91c"
).count()

if db_ds_objects > 0:
    print("Yes, one object with this condition exist")
```

It took **0.3 Seconds** to execute this block. Not bad right? But wait. Here count is a redundant part, we wanted to know if there exists only one instance where our condition satisfies. That's where **.exist()** method helps.

```py
db_ds_objects = Benchmark.objects.filter(
    databook_id="61722a62-fe71-44df-86a5-477bcdfbd91c"
).exists()

if db_ds_objects:
    print("Yes, this object exist")
```

Does the same thing as the previous block at the expense of **0.03 Seconds**. That's a whopping 100% or 1000% reduction in runtime. I know I'm bad at math :confused:

You may ask *"with the modern Machines I have, why should I even care about this bro?"*. The thing is once the data grows the time difference would increase exponentially.

> Things to Note:
> 1. It's not only about doing it on the database but also doing it in the best and right possible way.
> 2. Read Documentation. 

## 3. Call Q() when in trouble.

Let's say, we wanted to get the count of the objects which satisfies the following condition.

    (databook_id='A' OR datasheet_id='B') AND (NOT the records which has databook_id='A' and datasheet_id='B')

```py
databook_condition_objs = Benchmark.objects.filter(databook_id="A")
datasheet_condition_objs = Benchmark.objects.filter(datasheet_id="B")
combined_objects_before_check = []

for each_object in databook_condition_objs:
    combined_objects_before_check.append(each_object)

for each_object in datasheet_condition_objs:
    combined_objects_before_check.append(each_object)

combined_objects_after_check = []

for each_object in combined_objects_before_check:
    if not (each_object.databook_id == "A" and each_object.datasheet_id == "B"):
        combined_objects_after_check.append(each_object)

print(len(combined_objects_after_check))
```

Double query and filter: we can do two queries separately, iterate through the objects and filter for the **NOT** condition. But this is super expensive and very inefficient.

> Performance can be found even in the darkest of times, when one only remembers to read the documentation. - Not Dumbledore.

That's when Q() comes to the rescue. What it does is, it allows us to do complex queries at ease. Q() expression can be coupled with filter() function as well. 

**&, |, ~ symbols uses for AND, OR, NOT respectively.**


```py
databook_condition = Q(databook_id="A")
datasheet_condition = Q(datasheet_id="B")
both_condition = Q(databook_id="A") & Q(
    datasheet_id="B"
)

combined_objects_after_check = Benchmark.objects.filter(
    (databook_condition | datasheet_condition) & ~both_condition
).count()
print(combined_objects_after_check)

```

What did we get from the Q()?
1. Better Performance.
2. Better Code readability
3. Some Happiness.

## 4. What The F().

Now, let's say you have to update *"knowledge_end_date"* of all the rows which has databook_id = 'A'. And you have to update it as **"knowlege_begin_date" + 12 days**
Let me give an example:

Before Update:

| knowledge_begin_date | knowledge_end_date | databook_id | ... |
| ------ | ----------- | ----------- | ----------- |
| 2022-05-13   | NULL | A | ...|
| 2021-01-01 | NULL | B | ...|
| 2022-04-05    | NULL | A | ...|

After Update:

| knowledge_begin_date | knowledge_end_date | databook_id | ... |
| ------ | ----------- | ----------- | ----------- |
| 2022-05-13   | 2022-05-25 | A | ...|
| 2021-01-01 | NULL | B | ...|
| 2022-04-05    | 2022-04-17 | A | ...|


Bruh? You read this long?. Damn your attention span.

One thing that would come to mind is to fetch all the objects which have databook_id='A', iterate over them, update the *knowledge_end_date* and call *.save()*  method on the object. Below is the implementation of it.

```py
objects = Benchmark.objects.filter(datasheet_id="A")
for each_object in objects:
    each_object.knowledge_end_date = (
        each_object.knowledge_begin_date + timedelta(days=10)
    )
    each_object.save()
```

This triggers N+1 queries to update N objects. yeah yeah, I hear you, we can do bulk_update() but what If I tell you there is a better way.

```py
Benchmark.objects.filter(
    datasheet_id="A",
).update(knowledge_end_date=F("knowledge_begin_date") + timedelta(days=10))
```

This code does the same thing but better. What happens here is F() expression will do the modification within the table without fetching it. Note that, we can only add two things same type. Date + Date or Int + Int and so on. If you try to add Int with Date or String. F() expression will raise an exception.

> Things to Note:
> 1. Again, Make the Database do the hard work instead of Python.
> 2. F() reduces the number of queries and avoids race condition problem, which occurs in the case of iterate and save.

## 5. Iterate, Iterate, Iterate ...

Suppose your table has grown enormously large. You got into a situation where you need to fetch almost 90% of the data from the table. Next thing, you fire a single query with that condition and blow up your memory.

```py
databook_id = set()
benchmark_objects = Benchmark.objects.values("databook_id")
for obj in benchmark_objects:
    databook_ids.add(obj["databook_id"])
```

Cool, Now that we have blown up our memory with the above query. Let's go to .iterator()


An iterator is something that gives you only one instant object at any point in time. You cannot revisit the object you have seen before.

Django supports .iterator() which opens a database connection once and instead of fetching all things at one go, you fetch the objects chunk by chunk. you consume the first chunk of objects and then you move to the next chunk.

Yes, we are increasing the number of queries here for the advantage of doing things with minimal memory.

```py

databook_ids = set()
benchmark_objects = (
    Benchmark.objects.all().values("databook_id").iterator(chunk_size=2000)
)
for obj in benchmark_objects:
    databook_ids.add(obj["databook_id"])
```

Let's say there are **5420 Objects** in your table, **3 Queries** will be fired by the above implementation. 

**chunk_size** parameter helps us to configure, how many objects we need to pick for a query.

> Things to Note:
> 1. **.iterator()** increases the number of queries you will make but reduces memory drastically.
> 2. If the roundtrip (time takes to connect to the remote database, fetch data, and return it to you.) is high, consider increasing the chunk_size.


## 6. index good. scan bad.

There is no better magic in the whole database than Indexes. It's the true computer science marvel. And the easiest place to increase your application performance exponentially. 
Suppose if you have index like this, 

```py
indexes = [
    models.Index(
        fields=[
            "client_id",
            "-knowledge_end_date",
        ]
    ),
    models.Index(fields=["client_id", "databook_id", "datasheet_id"]),
]
```


Let's see which Queries will do Index Scan and which Queries won't.

```py
.filter(client_id='1') # Index Scan
.filter(client_id='1', databook_id='A') # Index Scan
.filter(databook_id='A') # Nopee, Sorry. 
.filter(datasheet_id='B') # Nopee, Sorry. 
.filter(client_id='1', databook_id='A',datasheet_id='B') # Index Scan
.filter(client_id='1', datasheet_id='B') # Nope, Sorry.
```

> Things to Note:
> 1. Try to use conditions in the filter in the same order as the Index.
> 2. Read https://use-the-index-luke.com/. No better site on the Internet explains Indexes like this.

That's it, folks. Thanks for reading till the end! *Next time!*

<div class="tenor-gif-embed" data-postid="12985913" data-share-method="host" data-aspect-ratio="1.785" data-width="50%"><a href="https://tenor.com/view/the-office-bow-michael-scott-steve-carell-office-gif-12985913">The Office Bow GIF</a>from <a href="https://tenor.com/search/the+office-gifs">The Office GIFs</a></div> <script type="text/javascript" async src="https://tenor.com/embed.js"></script>



 
