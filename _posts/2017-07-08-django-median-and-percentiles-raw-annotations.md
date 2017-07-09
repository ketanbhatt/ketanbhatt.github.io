---
title:	"Hack Django ORM to calculate Median and Percentiles (Or make annotations great again!)"
date:	2017-07-08
excerpt: "Hack Django ORM to make RawSQL work for you"
---
_originally posted on [medium][original-article-link]_

Hacks should be quick. And so should be the articles about them.

### Problem
We needed to calculate medians, percentiles for some quantities for our own ETL system _(note to self: write a post on this)_ grouped by certain fields.

Some options we had:
1. [Extra][django-extra]: But Django says this will be deprecated, and use it as a last resort. We still had resorts to explore.
2. [Raw SQL][django-raw-sql]: _Besides_ all usual bad things with writing RAW SQL (look at the number of warnings on the linked page!), the code starts to look ugly.

So what was the best way to do it?
Come on! Django also gives us something called a [**RawSQL**][django-rawsql]. Great. So we can just use it to get the percentiles we wanted. Right?

**Wrong**. As we realised later, RawSQL is better suited for aggregates and not annotations. Exhibit:

{% highlight python %}
q = MyModel.objects.values('some_fk_id').annotate(
    avg_duration=Avg('duration'),
    perc_90_duration=RawSQL('percentile_disc(%s) WITHIN GROUP (ORDER BY duration)', (0.9,)),
)

print q.query

# SELECT "some_fk_id",
#        AVG("duration") AS "avg_duration",
#        (percentile_disc(0.9) WITHIN
#         GROUP (
#                ORDER BY duration)) AS "perc_90_duration"
# FROM "mymodel"
# GROUP BY "some_fk_id", (percentile_disc(0.9) WITHIN
#                         GROUP (
#                                ORDER BY duration))
{% endhighlight %}

Notice how our Raw expression `percentile_disc(0.9) WITHIN GROUP (ORDER BY duration)` also gets added to the `GROUP BY` clause?

This would not happen if we remove the `Avg("duration")` from annotation. So basically, if the query already has a `GROUP BY` clause, `RawSQL` will add the `sql` to the `GROUP BY` clause as well.

This is not what we want. It also didn’t make sense to us, **why is that needed?** Maybe when we want to use `RawSQL` in an `order_by` and want the expression to get added to `GROUP BY` automatically? _Maybe_.

### Solution
We dug deep as to why is the sql added to `GROUP BY`. Looked at the [source code][django-rawsql-source], found this method `get_group_by_cols` which returns `self`. Super sensible naming by Django devs. I knew we could do something here. Ergo, the Hack:

{% highlight python %}
class RawAnnotation(RawSQL):
    """
    RawSQL also aggregates the SQL to the `group by` clause which defeats the purpose of adding it to an Annotation.
    """
    def get_group_by_cols(self):
        return []

# The Query
q = MyModel.objects.values('some_fk_id').annotate(
    avg_duration=Avg('duration'),
    perc_90_duration=RawAnnotation('percentile_disc(%s) WITHIN GROUP (ORDER BY duration)', (0.9,)),
)

print q.query

# SELECT "some_fk_id",
#        AVG("duration") AS "avg_duration",
#        (percentile_disc(0.9) WITHIN
#         GROUP (
#                ORDER BY duration)) AS "perc_90_duration"
# FROM "mymodel"
# GROUP BY "some_fk_id"
{% endhighlight %}

**We created a class `RawAnnotation` and overrode `get_group_by_cols` to return an empty array.** And now it works as expected.

Yay.

[original-article-link]: https://medium.com/squad-engineering/hack-django-orm-to-calculate-median-and-percentiles-or-make-annotations-great-again-23d24c62a7d0
[django-extra]: https://docs.djangoproject.com/en/1.11/ref/models/querysets/#extra
[django-raw-sql]: https://docs.djangoproject.com/en/1.11/topics/db/sql/
[django-rawsql]: https://docs.djangoproject.com/en/1.11/ref/models/expressions/#raw-sql-expressions
[django-rawsql-source]: https://docs.djangoproject.com/en/1.11/_modules/django/db/models/expressions/#RawSQL


