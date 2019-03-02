# The expression language

With more metrics collected by your Prometheus server, it is time to
familiarize yourself a bit more with the expression language. For comprehensive
documentation, check out the
[querying chapter](http://prometheus.io/docs/querying/basics/). The following
is meant as an inspiration for how to play with the metrics currently collected
by your server. Evaluate them in the *Console* and *Graph* tab. For the latter,
try different time ranges and the *stacked* option.

## The `rate()` function
Prometheus internally organizes sample data in chunks. It performs a number of
different chunk operations on them and exposes them as
`prometheus_local_storage_chunk_ops_total`, which is comprised of a number of
counters, one per possible chunk operation. To see a rate of chunk operations
per second, use the rate function over a time range that should cover at least
a handful of scrape intervals.

```
rate(prometheus_local_storage_chunk_ops_total[1m])
```

Now you can see the rate for each chunk operation type.  Note that the rate
function handles counter resets (for example if a binary is restarted).
Whenever a counter goes down, the function assumes that a counter reset has
happened and the counter has started counting from `0`.

## The `sum` aggregation operator
If you want to get the total rate for all operations, you need to sum up the
rates:

```
sum(rate(prometheus_local_storage_chunk_ops_total[1m]))
```

Note that you need to take the sum of the rate, and not the rate of the sum.
(Exercise for the reader: Why?)

## Select by label
If you want to look only at the persist operation, you can filter by label with
curly braces:

```
rate(prometheus_local_storage_chunk_ops_total{type="persist"}[1m])
```

You can use multiple label pairs within the curly braces (comma-separated), and
the match can be inverted (with `!=`) or performed with a regular expression
(with `=~`, or `!~` for the inverted match).

(Exercise: How to estimate the average number of samples per chunk?)

## Aggregate by label
The metric `http_request_duration_microseconds_count` counts the number of HTTP
requests processed. (Disregard the `duration_microseconds` part for now. It
will be explained later.) If you look at it in the *Console* tab, you can see
the many time series with that name. The metric is partitioned by handler,
instance, and job, resulting in many sample values at any given time. We call
that an instant vector.

If you are only interested in which job is serving how many QPS, you can let
the sum operator aggregate by job (resulting in the two jobs we are monitoring,
the Prometheus itself and the node exporter):

```
sum(rate(http_request_duration_microseconds_count[5m])) by (job)
```

A combination of label pairs is possible, too. You can aggregate by job and
instance (which is interesting if you have added an additional node exporter to
your config):

```
sum(rate(http_request_duration_microseconds_count[5m])) by (job, instance)
```

Note that there is an alternative syntax with the `by` clause following
directly the aggregation operator. This syntax is particularly useful in
complex nested expressions, where it otherwise becomes difficult to spot which
`by` clause belongs to which operator.

```
sum by (job, instance) (rate(http_request_duration_microseconds_count[5m]))
```

## Arithmetic
There is a metric `http_request_duration_microseconds_sum`, which sums up the
duration of all HTTP requests. If the labels match, you can easily divide two
instant vectors, yielding the average request duration in this case:

```
rate(http_request_duration_microseconds_sum[5m]) / rate(http_request_duration_microseconds_count[5m])
```

You can aggregate as above if you do it separately for numerator and
denominator:

```
sum(rate(http_request_duration_microseconds_sum[5m])) by (job) / sum(rate(http_request_duration_microseconds_count[5m])) by (job)
```

Things become more interesting if the labels do not match perfectly
between two instant vectors or you want to match vector elements in a
many-to-one or one-to-many fashion. See the
[vector-matching section](http://prometheus.io/docs/querying/operators/#vector-matching)
in the documentation for details.

## Summaries
Rather than an average request duration, you will be more often interested in
quantiles like the median or the 90th percentile. To serve that need,
Prometheus offers summaries. `http_request_duration_microseconds` is a summary
of HTTP request durations, and `http_request_duration_microseconds_sum` and
`http_request_duration_microseconds_count` are merely byproducts of that
summary.  If you look at `http_request_duration_microseconds` in the expression
browser, you see a multitude of time series, as the metric is now partitioned
by quantile, too. An expression like
`http_request_duration_microseconds{quantile="0.9"}` displays the 90th
percentile request duration. You might be tempted to aggregate the result as
you have done above. Not possible, unfortunately. Welcome to the wonderland of
statistics.

Read more about
[histograms and summaries](http://prometheus.io/docs/practices/histograms/)
in the documentation.

## Recording rules
In your practical work with Prometheus at scale, you will pretty soon run into
expressions that are very expensive and slow to evaluate. The remedy is
*recording* rules, a way to tell Prometheus to pre-calculate expressions,
saving the result in a new time series, which can then be used instead of the
expensive expression. See the documentation for details:
* [General documentation about rules](http://prometheus.io/docs/querying/rules/).
* [Best practices for naming rules](http://prometheus.io/docs/practices/).
