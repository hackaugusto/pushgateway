# Prometheus Pushgateway

The Prometheus pushgateway exists to allow ephemeral and batch jobs to
expose their metrics to Prometheus. Since these kinds of jobs may not
exist long enough to be scraped, they can instead push their metrics
to a pushgateway. The pushgateway then exposes these metrics to
Prometheus.

The pushgateway is explicitly not an aggregator, but rather a metrics
cache. It does not have a statsd-like semantics. The metrics pushed
are exactly the same as you would present for scraping in a
permanently running program.

## Run it

Compile the binary as usual (`go get`, `go build`). It's pure Go, no
dependancy on non-Go libraries.

For the most basic setup, just start the binary. To change the address
to listen on, use the `-addr` flag. The `-persistence.file` flag
allows you to specify a file in which the pushed metrics will be
persisted (so that they survive restarts of the pushgateway).

## Use it

### Libraries

Prometheus client libraries should have a feature to push the
registered metrics to a pushgateway. Usually, a Prometheus client
passively presents metric for scraping by a Prometheus server. A
client library that supports pushing has a push function, which needs
to be called by the client code. It will then actively push the
metrics to a pushgateway, using the API described below.

### Command line

Using the Prometheus text protocol, pushing metrics is so easy that no
separate CLI is provided. Simply use a command-line HTTP tool like
`curl`. Your favorite scripting language has most likely some built-in
HTTP capabilities you can leverage here as well.

*Caveat: Note that in the text protocol, each line has to end with a
line-feed character (aka 'LF' or '\n'). Ending a line in other ways,
e.g. with 'CR' aka '\r', 'CRLF' aka '\r\n', or just the end of the
packet, will result in a protocol error.*

Examples:

* Push a single sample:

        echo "some_metric 3.14" | curl --data-binary @- http://pushgateway.example.org:8080/metrics/jobs/some_job

  Since no type information has been provided, `some_metric` will be of type `untyped`.

* Push something more complex:

        cat <<EOF | curl --data-binary @- http://pushgateway.example.org:8080/metrics/jobs/some_job/instances/some_instance
        # TYPE some_metric counter
        some_metric{label="val1"} 42
        # This one even has a timestamp (but beware, see below).
        some_metric{label="val2"} 34 1398355504000
        # TYPE another_metric gauge
        # HELP another_metric Just an example.
        another_metric 2398.283
        EOF

  Note how type information and help strings are provided. Those lines
  are optional, but strongly encouraged for anything more complex.

* Delete all metrics of an instance:

        curl -X DELETE http://pushgateway.example.org:8080/metrics/jobs/some_job/instances/some_instance

* Delete all metrics of a job:

        curl -X DELETE http://pushgateway.example.org:8080/metrics/jobs/some_job

### About timestamps

If you push metrics at time *t<sub>1</sub>*, you might be tempted to
believe that Prometheus will scrape them with that same timestamp
*t<sub>1</sub>*. Instead, what Prometheus attaches as a timestamp is
the time when it scrapes the push gateway. Why so?

In the world view of Prometheus, a metric can be scraped at any
time. A metric that cannot be scraped has basically ceased to
exist. Prometheus is somewhat tolerant, but if it cannot get any
samples for a metric in 5min, it will behave as if that metric does
not exist anymore. Preventing that is actually one of the reasons to
use a push gateway. The push gateway will make the metrics of your
ephemeral job scrapable at any time. Attaching the time of pushing as
a timestamp would defeat that purpose because 5min after the last
push, your metric will look as stale to Prometheus as if it could not
be scraped at all anymore. (Prometheus knows only one timestamp per
sample, there is no way to distinguish a 'time of pushing' and a 'time
of scraping'.)

You can still force Prometheus to attach a different timestamp by
using the optional timestamp field in the exchange format. However,
there are very few use cases where that would make
sense. (Essentially, if you push more often than every 5min, you
could attach the time of pushing as a timestamp.) 

## API

All pushes are done via HTTP. The interface is REST-like.

### URL

The default port the push gateway is listening to is 8080. The path looks like

    /metrics/jobs/<JOBNAME>[/instances/<INSTANCENAME>]

`<JOBNAME>` is used as the value of the `job` label, and `<INSTANCE>`
as the value of the `instance` label. The instance part of the URL is
optional. If it is missing, the IP number of the pushing host is used
as the value for the 'instance' label instead.

If those labels are already set in the body of the request (as regular
labels, e.g. `name{job="foo",instance="bar"} 42`), _the values of
those labels will be overwritten with values determined as described
above!_ (This behavior might be changed in the future if a valid
use-case can be shown.)

### `PUT` / `POST` method

`PUT` and `POST` are treated the same for the sake of simplicity (in
violation of HTTP conventions).  They are both used to push metrics,
and the response code upon success is always 202 (even if that same
metric has never been pushed before, i.e. there is no feedback to the
client if the push has replaced an metric or created a new one).

_Note that each push will completely replace all existing metrics with
the same name, job, and instance combination as the metrics pushed,
even if the existing metrics had a different label set otherwise._

The body of the request contains the metrics to push either as delimited binary protocol
buffers or in the simple flat text format (both in version 0.0.4, see the
[data exposition format specification](https://docs.google.com/document/d/1ZjyKiKxZV83VI9ZKAXRGKaUKK2BIWCT7oiGBKDBpjEY/edit?usp=sharing). Discrimination between the two variants is done via content-type
header. (In case of an unknown content-type, the text format is tried as a fall-back.)

_If using the protobuf format, do not send duplicate MetricFamily
proto messages (i.e. more than one with the same name) in one push, as
they will overwrite each other._

A 204 response to the request means that the pushed metrics are queued
for an update of the storage. Scraping the push gateway may still
yield the old sample value for that metric (or nothing at all if this
is the first time that metric is pushed). Neither is there a guarantee
that the metric is persisted to disk. (A server crash may cause data
loss. Or the push gateway is configured to not persist to disk at all.)

### `DELETE` method

`DELETE` is used to delete metrics from the push gateway. The request
must not contain any content. If both a job and an instance are
specified in the URL, all metrics matching that job and instance are
deleted. If only a job is specified, all metrics matching that job are
deleted. The response code upon success is always 202. The delete
request is merely queued at the moment. There is no guarantee that the
request will actually be executed or that the result will make it to
the persistence layer (e.g. in case of a server crash). However, the
order of `PUT`/`POST` and `DELETE` request is guaranteed, i.e. if you
have successfully sent a `DELETE` request and then send a `PUT`, it is
guaranteed that the `DELETE` will be processed first (and vice versa).

Deleting non-existing metrics is a no-op and will not result in an
error.