---
title:	"ElasticSearch, Django and Haystack"
date:	2016-04-19
excerpt: How we used ElasticSearch and Haystack to power searches on our Django Admin. Documented here are our trials and learnings with Haystack. And a cool new fork for you to use.
---
**TLDR;** Use [squadrun/django-haystack](https://github.com/squadrun/django-haystack) and live your life happily. Also always index in batches. Also read the whole of it.

A problem our Operations Team at [SquadRun]() was facing was how slow the searches on the Admin panel used to work. It wasn't surprising though. We had around _30 Million to 8 Hundred Thousand rows_ in different models. Now that is not a lot of data, but it is certainly a lot if you want to search on a lot of text and non-text fields simultaneously. We also didn't want our database to spend resources on the search queries. At a time when the database is already under load, these searches would:

1. Return late
2. Use even more resources of the db already under load

We decided this needs to stop as it kills the team's effectiveness. So we planned to index those models which are frequently searched in an ElasticSearch (ES) server. We also needed a way to integrate ES with the django admin (because we do searches only from the admin). We had two options:

1. Custom implementation for the search, and using ES APIs directly
2. Something like [Haystack](https://github.com/django-haystack/django-haystack) which handles all of this for you.

It is advisable for you to directly deal with ES's API if you have something complex in your mind, or if you are an expert with ES (but then you wouldn't have been reading this). If your goal is just to do away with admin searches hitting your database, and want to plug in ES into the flow, I suggest haystack will serve you well. 


## Is that it?
No, haystack hasn't been updated in a while ([121](https://github.com/django-haystack/django-haystack/pulls) and [320](https://github.com/django-haystack/django-haystack/issues) open PRs and issues as of 22-04-2016) and thus lacks a lot of features. Here are the problems we faced:

1. Updating index every time a row is added/updated is time taking, and sometimes updates happen in bulk (`MyModel.objects.filter(old_stuff=True).update(old_stuff=False)`), and this won't get updated in the index automatically. [Solution](#updating-index-asynchronously-and-indexing-everything-else-that-was-left)
2. Defining what fields to index using templates takes me away from code, and every time I want to find what fields I am indexing, I will have to dig out the template file for the model. Cumbersome. [Solution](#make-haystack-templates-obsolete-and-faster-indexing)
3. Indexing was slow if you were indexing fields across tables. [Solution](#make-haystack-templates-obsolete-and-faster-indexing)
4. **No filter and ordering support** :( [Solution](#fix-filtering-and-ordering)
5. [Django haystack EdgeNgramField gives different results than elasticsearch](http://stackoverflow.com/questions/20430449/django-haystack-edgengramfield-given-different-results-than-elasticsearch). [Solution](#solve-problem-i-cant-seem-to-find-a-name-for-and-modify-tokens-length)
6. The `edgengram` tokenizer and filter's `min_gram` and `max_gram` were in the range 3-15, that meant words like `a, an, of` won't get indexed and a search for `King of Nepal` will not return anything because `of` is not in the index and so there is no match. We could ask our team to eliminate such words from their searches, but that is not just it is supposed to be done, we wanted to make their life easy, and not make them remember new rules. [Solution](#solve-problem-i-cant-seem-to-find-a-name-for-and-modify-tokens-length)


## All I see is complaints
Me too, here is what we did to solve them:

### Updating Index asynchronously And indexing everything else that was left
This one is a no-brainer and is even suggested in haystack's docs. Use [celery-haystack](http://celery-haystack.readthedocs.org/en/latest/) to update index asynchronously. 
(Gotcha! --> add `CELERY_HAYSTACK_COUNTDOWN = 2` to your settings, otherwise you are going to get a lot of `DoesNotExist` errors)

So celery-haystack works by catching save/delete signals that django throws. Cool? Not yet. 

What about updates/creates that happen in bulk? No signals are emitted for those and that would mean those things will never be indexed. This is not ideal. You can not stop updating/creating in bulk because of this.
So, we decided to run a cron every 10 minutes that reindexes everything that was created/updated in the last 10 minutes.

How did we know what happened in the last 10 minutes? We have an `updated_at` field in the models we are indexing, so that combined with haystack's [`update_index`](http://django-haystack.readthedocs.org/en/v2.4.1/management_commands.html#update-index) management command and its `age` parameter, we were able to achieve it. 
Another problem. When we do a `queryset.update()` in django, `auto_now` fields are not updated. Now what? Now a little hack :D

{% highlight python %}
def update_with_last_modified_time(qs, **kwargs):
    # This function adds any auto_now field to the update call because QuerySet.update() doesn't do it :X
    model_fields = qs.model._meta.get_fields()
    fields_and_value_map = {}
    for field in model_fields:
        try:
            auto_now = field.__getattribute__('auto_now')
        except AttributeError:
            auto_now = False

        if auto_now:
            if type(field) == DateField:
                fields_and_value_map[field.name] = datetime.date.today()
            elif type(field) == DateTimeField:
                fields_and_value_map[field.name] = timezone.now()

    fields_and_value_map.update(kwargs)
    return qs.update(**fields_and_value_map)
{% endhighlight %}

We use this function wherever we are doing a `queryset.update()`. So this is solved.


### Make haystack templates obsolete and faster indexing
1. Need to index faster --> Use `select_related` while fetching objects to index
2. Track indexed fields in code --> Get inspired by django admin's implementation of `readonly_fields`

So we coded a mixin:
{% highlight python %}
class AutoPrepareTextIndexMixin(object):
    """
    Used with Indexed Classes to add common functions:
    1. Prepares text to be documented using fields from the list document_fields
    2. Gets fields to be `select_related` for efficient db querying while indexing
    3. Defines "get_updated_field"
    Also tries to check for these modifications and raises errors if things not implemented like they should be
    """

    def __init__(self):
        super(AutoPrepareTextIndexMixin, self).__init__()
        default_pk_field = getattr(settings, 'HAYSTACK_ADMIN_DEFAULT_ORDER_BY_FIELD', None)
        if default_pk_field:
            if default_pk_field not in self.fields.keys():
                raise NotImplementedError('Need a field for "{0}" in {1} like so "{0} = indexes.'
                                          'IntegerField(model_attr=\'pk\')"'.format(default_pk_field, self.__class__))

        if self.get_content_field() != 'text':
            raise NameError('The content field should be named "text" in {} or it wont be '
                            'prepared like we want it to'.format(self.__class__))

        document_fields = getattr(self, 'document_fields', None)
        if not document_fields:
            raise NotImplementedError('"document_fields" not specified for {}, what am I going to put in '
                                      'the document brah?'.format(self.__class__))

    def get_updated_field(self):
        return 'updated_at'

    def index_queryset(self, using=None):
        select_related_for_index = getattr(self, 'select_related_for_index', None)
        if select_related_for_index:
            return self.get_model().objects.select_related(*self.select_related_for_index)
        else:
            return self.get_model().objects.all()

    def prepare_text(self, obj):
        document_fields = getattr(self, 'document_fields', None)

        values = []
        for field in document_fields:
            full_field_list = field.split('__')
            field_value = reduce(lambda acc_attr, attr: getattr(acc_attr, attr), full_field_list, obj)
            values.append(field_value)

        return ' '.join(map(unicode, values))
{% endhighlight %}

Self explanatory. Add this mixin to your search indexes classes. Also implements `get_updated_field` because I didn't want to copy this code everywhere.

### Solve problem I can't seem to find a name for and modify token's length
For that, we overrode the `ElasticsearchSearchEngine` class and came up with:
{% highlight python %}
class StandardAnalyzerElasticBackend(ElasticsearchSearchBackend):
    SET_ANALYZE_STANDARD_FOR_SEARCH = getattr(settings, 'SET_ANALYZE_STANDARD_FOR_HAYSTACK_SEARCH', False)
    DEFAULT_SETTINGS = {
        'settings': {
            "analysis": {
                "analyzer": {
                    "ngram_analyzer": {
                        "type": "custom",
                        "tokenizer": "standard",
                        "filter": ["haystack_ngram", "lowercase"]
                    },
                    "edgengram_analyzer": {
                        "type": "custom",
                        "tokenizer": "standard",
                        "filter": ["haystack_edgengram", "lowercase"]
                    }
                },
                "tokenizer": {
                    "haystack_ngram_tokenizer": {
                        "type": "nGram",
                        "min_gram": 3,
                        "max_gram": 15,
                    },
                    "haystack_edgengram_tokenizer": {
                        "type": "edgeNGram",
                        "min_gram": 1,
                        "max_gram": 26,
                        "side": "front"
                    }
                },
                "filter": {
                    "haystack_ngram": {
                        "type": "nGram",
                        "min_gram": 3,
                        "max_gram": 15
                    },
                    "haystack_edgengram": {
                        "type": "edgeNGram",
                        "min_gram": 1,
                        "max_gram": 26
                    }
                }
            }
        }
    }

    def build_search_kwargs(self, query_string, **kwargs):
        search_kwargs = super(StandardAnalyzerElasticBackend, self).build_search_kwargs(query_string, **kwargs)
        if self.SET_ANALYZE_STANDARD_FOR_SEARCH:
            try:
                search_kwargs['query']['filtered']['query']['query_string']['analyzer'] = 'standard'
            except KeyError:
                pass

        return search_kwargs


class StandardAnalyzerElasticSearchEngine(ElasticsearchSearchEngine):
    backend = StandardAnalyzerElasticBackend
{% endhighlight %}

Now plug this Backend for your `ENGINE` key in `HAYSTACK_CONNECTIONS` settings,and you are golden.


### Fix filtering and ordering 
Yo, Done! Read on to know more.


## So do I copy all of this into my code?
No, you can just use [squadrun/django-haystack](https://github.com/squadrun/django-haystack) and get all these things done for you (even fixed the filtering!). Go on, check the [Diff between the original and this fork](https://github.com/django-haystack/django-haystack/compare/master...squadrun:master) if you don't trust me.

This is how my indexes look like:
{% highlight python %}
class MyModelIndex(AutoPrepareTextIndexMixin, CelerySearchIndex, indexes.Indexable):
    model_pk = indexes.IntegerField(model_attr='pk')  # This is required
    text = indexes.EdgeNgramField(document=True)  # This too
    some_boolean = indexes.IntegerField(model_attr='some_boolean')

    # Filters should map to the exact field name that the admin will access them by.
    # Example: foreign keys are accessed by FKModel__id. This is for filters
    rel_model = indexes.IntegerField(model_attr='rel_model__id')

    document_fields = ['name', 'nickname', 'rel_model__title']  # ^_^
    select_related_for_index = ['business']

    def get_model(self):
        return MyModel
{% endhighlight %}

See any mistakes in the fork, send a PR. Want to do similar things and solve problems, send an Email!