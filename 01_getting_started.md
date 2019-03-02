# Getting started

## Getting Prometheus
Download the latest binary release of Prometheus for your platform from:

https://github.com/prometheus/prometheus/releases

Extract the contents into a new directory and change to that directory.

```

```

## Configuring Prometheus to monitor itself

Take a look at the included example `prometheus.yml` configuration file. It
configures global options, as well as a single job to scrape metrics from: the
Prometheus server itself.

Prometheus collects metrics from monitored targets by scraping metrics HTTP
endpoints on these targets. Since Prometheus also exposes data in the same
manner about itself, it may also be used to scrape and monitor its own health.
While a Prometheus server which collects only data about itself is not very
useful in practice, it is a good starting example.

## Starting Prometheus
Start Prometheus. By default, Prometheus reads its config from a file
called `prometheus.yml` in the current working directory, and it
stores its database in a sub-directory called `data`, again relative
to the current working directory. Both behaviors can be changed using
the flags `-config.file` or `-storage.local.path`, respectively.

```

```

Prometheus should start up and it should show the targets it scrapes at
[http://localhost:9090/targets](http://localhost:9090/targets). You
will find [http://localhost:9090/metrics](http://localhost:9090/metrics) in the
list of scraped targets. Give Prometheus a couple of seconds to start
collecting data about itself from its own HTTP metrics endpoint.

You can also verify that Prometheus is serving metrics about itself by
navigating to its metrics exposure endpoint:
[http://localhost:9090/metrics](http://localhost:9090/metrics).

## Using the expression browser
The query interface at
[http://localhost:9090/](http://localhost:9090/) allows you to
explore metric data collected by the Prometheus server. At the moment, the
server is only scraping itself. The collected metrics are already quite
interesting, though.  The *Console* tab shows the most recent value of metrics,
while the *Graph* tab plots values over time. The latter can be quite expensive
(for both the server and the browser). It is in general a good idea to try
potentially expensive expressions in the *Console* tab first. Take a bit of
time to play with the expression browser. Suggestions:

* Evaluate `prometheus_local_storage_ingested_samples_total`, which shows you
  the total number of ingested samples over the lifetime of the server. In the
  *Graph* tab, it will show as steadily increasing.
* The expression `prometheus_local_storage_ingested_samples_total[1m]`
  evaluates to all sample values of the metric in the last minute. It cannot be
  plotted as a graph, but in the *Console* tab, you see a list of the values with
  (Unix) timestamp.
* `rate(prometheus_local_storage_ingested_samples_total[1m])` calculates the
  rate (increase per second) over the 1m timeframe. In other words, it tells you
  how many samples per second your server is ingesting. This expression can be
  plotted nicely, and it will become more interesting as you add more targets.

## Start the node exporter
The node exporter is a server that exposes system statistics about the machine
it is running on as Prometheus metrics.

Download the latest node exporter binary release for your platform from:

https://github.com/prometheus/node_exporter/releases

Beware that the majority of the node exporter's functionality is
Linux-specific, so its exposed metrics will be significantly reduced when
running it on other platforms.

Linux example:

```

```

Start the node exporter:

```
./node_exporter
```

## Configure Prometheus to monitor the node exporter

If you are not running your local node exporter under Linux, you might want to
point your Prometheus server to a Linux node exporter run by one of your peers
in the workshop. Or point it to a node exporter we are running during the
workshop at
[http://demo.robustperception.io:9100/metrics](http://demo.robustperception.io:9100/metrics).

Add the following job configuration to the `scrape_configs:` section
in `prometheus.yml` to monitor both your own and the demo node
exporter:

```
  - job_name: 'node'
    scrape_interval: '15s'
    static_configs:
      - targets:
          - 'localhost:9100'
          - 'demo.robustperception.io:9100'
```

Send your Prometheus server a `SIGHUP` to initiate a reload of the configuration:

```
killall -HUP prometheus
```

Then check the *Status* page of your Prometheus server to make sure the node
exporter is scraped correctly. Shortly after, a whole lot of interesting
metrics will show up in the expression browser, each of them starting with
`node_`. (Reload the page to see them in the autocompletion.) As an example,
have a look at `node_cpu`.

The node exporter has a whole lot of modules to export machine
metrics. Have a look at the
[README.md](https://github.com/prometheus/node_exporter) to get an
idea. While Prometheus is particularly good at collecting service
metrics, correlating those with system metrics from individual
machines can be immensely helpful.  (Perhaps that one task that showed
high latency yesterday was scheduled on a node with a lot of competing
disk operations?)

## Use the node exporter to export the contents of a text file
The *textfile* module of the node exporter can be used to expose static
machine-level metrics (such as what role a machine has) or the outcome of
machine-tied batch jobs (such as a Chef client run). To use it, create a
directory for the text files to export and (re-)start the node exporter with
the `-collector.textfile.directory` flag set. Finally, create a text file in
that directory.

```
mkdir textfile-exports
./node_exporter --collector.textfile.directory=textfile-exports
echo 'role{role="workshop_node_exporter"} 1' > textfile-exports/role.prom.$$
mv textfile-exports/role.prom.$$ textfile-exports/role.prom
```

For details, see the
[documentation](https://github.com/prometheus/node_exporter#textfile-collector).

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
