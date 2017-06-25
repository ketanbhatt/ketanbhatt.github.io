---
title:	"Blazingly fast querying on huge tables by avoiding joins"
date:	2017-05-14
excerpt: "Tl;dr: Avoid joins on large tables and evaluate parts of queries beforehand to get 100–10,000x performance gains!"
---
_originally posted on [medium][original-article-link]_

**Tl;dr**: Avoid joins on large tables and evaluate parts of queries beforehand to get **100–10,000x performance gains!**

As mentioned in a [previous post]({% post_url 2017-04-19-estimated-counts-for-faster-django-admin-change-list %}), because of some of our tables growing in size, our queries started performing poorly which resulted in a performance hit to our most used APIs. It was time we revisit some of these queries and do something that will give us the best possible outcome with the least effort.

### Diagnosis
Our old query _(that took 29 seconds to run)_ was something on the lines of:
{% highlight sql %}
select .. from .. inner join .. where (JOIN_PREDICATE);
{% endhighlight %}

We used `EXPLAIN ANALYSE` and [explain.depesz.com][depesz] to get an idea of the query that was being run. The reason our queries were running so slowly was:
1. In our case, there was a [Hash Join][hash-join] taking place, which would create a hash table from rows of one of the candidate tables which match the `join predicate`. Now this table can be quickly used for a lookup with the rows of the other candidate in the JOIN. **But** if we do this for two very large tables _(50m and 150m rows)_, it would mean a lot of memory being used up for the intermediate hash, as well as a lot of rows from the other candidate being looked up against this hash table.
2. Appropriate indices weren’t being used in the prepared queries. That could be due to [various reasons][indices-reason].

### Solution
Armed with the knowledge, **we thought that if we could just remove the `JOIN` from the query, it should return faster**.
We basically had to convert:
{% highlight sql %}
select .. from .. inner join .. where (JOIN_PREDICATE);
{% endhighlight %}
to:
{% highlight sql %}
select ... from .. where (column_value IN (1, 2, 3))
{% endhighlight %}
where `column_value IN (1, 2, 3)` is the result of the `JOIN_PREDICATE` ran separately before.

>
Our experiments showed us that there were huge performance gains. Our queries **went down from taking 29 seconds to a few milliseconds!**

### I don’t believe you
Let’s create two tables:
1. `User`
2. `Purchase`

Each `user` can have multiple `purchases`.

The code for creating the tables and inserting data is as follows:
{% highlight sql %}
-- User Table
CREATE TABLE user (
id serial PRIMARY KEY,
account_id int not NULL,
name varchar(10)
);

-- Purchase Table
CREATE TABLE purchase (
id SERIAL PRIMARY KEY,
data text not NULL,
user_id int not NULL
);

-- Populate our tables (might not be the most efficient way)
INSERT INTO USER (account_id)
SELECT generate_series(1,50000000) AS random_id;

INSERT INTO purchase (data, user_id)
SELECT 'Cereals' AS data,
       generate_series(1,50000000);
       
INSERT INTO purchase (data, user_id)
SELECT 'Milk' AS data,
       generate_series(1,50000000);


-- To mock a more real world example, we will add necessary indices
CREATE index ON "user" ("account_id");
CREATE index ON "purchase" ("user_id");
{% endhighlight %}

#### What is the query for?
We want to **get all the purchases for the given account IDs**.

#### Run 1: Join Query
{% highlight sql %}
SELECT "purchase"."id"
FROM "purchase"
INNER JOIN "user" ON ("purchase"."user_id" = "user"."id")
WHERE "user"."account_id" IN
    (SELECT generate_series(1,1000));
{% endhighlight %}

Here is the `EXPLAIN ANALYSE` output for this query: [https://explain.depesz.com/s/kGP](https://explain.depesz.com/s/kGP)

**Time taken: 100 seconds**

#### Run 2: Evaluate and Select
{% highlight sql %}
WITH user_ids AS
  (SELECT id
   FROM user
   WHERE account_id IN
       (SELECT generate_series(1,1000)))
SELECT purchase.id
FROM purchase
WHERE user_id IN
    (SELECT id
     FROM user_ids);
{% endhighlight %}

Here is the `EXPLAIN ANALYSE` output for this query: [https://explain.depesz.com/s/9dE](https://explain.depesz.com/s/9dE)

**Total Time taken: 7 ms**

#### Results
Join Query: 100 seconds

Evaluate and Select: 7milliseconds

**Performance Gain: 10,000x**

### Notes
1. Tested on `postgresql 9.6.2`
2. Huge gains only when the `join predicate` matches 100+ rows, otherwise performance will be more or less the same in both the cases.

[original-article-link]: https://medium.com/squad-engineering/blazingly-fast-querying-on-huge-tables-by-avoiding-joins-5be0fca2f523
[depesz]: https://explain.depesz.com/
[hash-join]: https://www.depesz.com/2013/05/09/explaining-the-unexplainable-part-3/#hash
[indices-reason]: https://www.depesz.com/2010/09/09/why-is-my-index-not-being-used/