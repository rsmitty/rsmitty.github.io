---
layout: post
comments: true
author: Spencer
title: A Noob's Guide to Custom Prometheus Exporters
tags: kubernetes golang prometheus
---

Recently I've been using prometheus at work to monitor and alert on the status of our Kubernetes clusters, as well as services we have running in the cluster. One really nice thing about using prometheus is that Kubernetes already exposes a `/metrics` endpoint and it's pretty simple to configure prometheus to scrape it. For other services, prometheus can even look for annotations on your pod definitions and begin scraping them automatically. However, not all software comes with snazzy prometheus endpoints built-in. As such, this post will go through the process of exposing your own endpoint and writing metrics out to it using golang.

## **The Basics** ##

It's important to learn a bit about the different pieces involved before we start stepping into the code. First, know that there's already a golang SDK for prometheus which makes the process quite nice. You can find that in github [here](https://github.com/prometheus/client_golang). An absolute high level of how prometheus does its thing is necessary as well. There's quite a few ways to deploy Prometheus on Kubernetes (which we won't deep dive on). We have been using the helm installation, which has been pretty straight forward. But once you've deployed it, it's mostly just making sure that you've got your pods configured with the proper annotations so that Promtheus picks them up automatically. Here's an example:

```yaml
...
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"
  prometheus.io/path: "/metrics" # this is the default already
...
```
Once these are added, Prometheus will automatically hit the `/metrics` endpoint and pull any info you expose there.

Finally, it helps to know a little bit about the different types of metrics in Prometheus. A great write-up is on the Prometheus site [here](https://prometheus.io/docs/concepts/metric_types/). For today, we'll be worrying about "counters". It's just a simple number value that always goes up. You'll likely find yourself using "gauges" pretty quickly as well, since they're effectively counters that go up or down.

## **The Code** ##

Alright, let's get some code down. First, we need a webserver. Golang does a great job of making this easy, but we're also going to import the `promhttp` library since it's necessary to handle the actual communication with prometheus.

- Create a `main.go` file in the subdirectory of your choice. Paste the following contents:

```go
package main

import (
  "net/http"

  log "github.com/Sirupsen/logrus"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
  //This section will start the HTTP server and expose
  //any metrics on the /metrics endpoint.
  http.Handle("/metrics", promhttp.Handler())
  log.Info("Beginning to serve on port :8080")
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

- Notice that there's a couple of imported packages. You may wish to install dep or something similar in order to `go get` these. I just used `dep init`.
- Once you've got the imports, this will actually function as expected right away! Issue `go run main.go` and issue `curl 127.0.0.1:8080/metrics`. You'll notice a significant amount of metrics already. The reason you'll see the metrics below is because the Prometheus package already exposes some basic info about the golang environment it's running in automatically.

```bash
# HELP go_gc_duration_seconds A summary of the GC invocation durations.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0
go_gc_duration_seconds{quantile="0.25"} 0
...
...
...
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.51986054859e+09
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 4.00805888e+08
```

It's now time to add our own metric, but there's a bit to understand about the flow of it as we go. At a high level what happens is that you implement a "collector", which is an interface provided by the Prometheus client. This collector is registered with the Prometheus client when your exporter starts up and the metrics you scrape are exposed to the metrics endpoint automatically.

Let's get it going:

- Create a new file called `collector.go` in the same directory.
- In the file, paste the following code. Notice that it's heavily documented with what's going on in the file.

```go
package main

import (
	"github.com/prometheus/client_golang/prometheus"
)

//Define a struct for you collector that contains pointers
//to prometheus descriptors for each metric you wish to expose.
//Note you can also include fields of other types if they provide utility
//but we just won't be exposing them as metrics.
type fooCollector struct {
	fooMetric *prometheus.Desc
	barMetric *prometheus.Desc
}

//You must create a constructor for you collector that
//initializes every descriptor and returns a pointer to the collector
func newFooCollector() *fooCollector {
	return &fooCollector{
		fooMetric: prometheus.NewDesc("foo_metric",
			"Shows whether a foo has occurred in our cluster",
			nil, nil,
		),
		barMetric: prometheus.NewDesc("bar_metric",
			"Shows whether a bar has occurred in our cluster",
			nil, nil,
		),
	}
}

//Each and every collector must implement the Describe function.
//It essentially writes all descriptors to the prometheus desc channel.
func (collector *fooCollector) Describe(ch chan<- *prometheus.Desc) {

	//Update this section with the each metric you create for a given collector
	ch <- collector.fooMetric
	ch <- collector.barMetric
}

//Collect implements required collect function for all promehteus collectors
func (collector *fooCollector) Collect(ch chan<- prometheus.Metric) {

	//Implement logic here to determine proper metric value to return to prometheus
	//for each descriptor or call other functions that do so.
	var metricValue float64
	if 1 == 1 {
		metricValue = 1
	}

	//Write latest value for each metric in the prometheus metric channel.
	//Note that you can pass CounterValue, GaugeValue, or UntypedValue types here.
	ch <- prometheus.MustNewConstMetric(collector.fooMetric, prometheus.CounterValue, metricValue)
	ch <- prometheus.MustNewConstMetric(collector.barMetric, prometheus.CounterValue, metricValue)

}
```

Walking through the file above, you'll notice that you must create an initializer, a "Describe" function, and a "Collect" function. These seem to be the bare minimum requirements. You'll also notice that we're creating two metrics, `fooMetric` and `barMetric` if you look in the `fooCollector` struct. The initializer does exactly what you'd expect, returns a pointer to the collector after adding some descriptions to `fooMetric` and `barMetric`. Describe simply writes those descriptions out to the channel that is passed in. Finally, Collect simply writes your desired metric values out to the channel that is passed in (simply `1` in our case).

- Update your `main.go` to register"fooCollector" when starting up. The whole `main.go` should look like the following:

```go
package main

import (
  "net/http"

  log "github.com/Sirupsen/logrus"
  "github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {

  //Create a new instance of the foocollector and 
  //register it with the prometheus client.
  foo := newFooCollector()
  prometheus.MustRegister(foo)

  //This section will start the HTTP server and expose
  //any metrics on the /metrics endpoint.
  http.Handle("/metrics", promhttp.Handler())
  log.Info("Beginning to serve on port :8080")
  log.Fatal(http.ListenAndServe(":8080", nil))
}
```

- Now, we can run the file just like before and see our new metrics exposed! Hit the `/metrics` endpoint after starting your webserver with `go run main.go collector.go`.

```bash
$ curl 127.0.0.1:8080/metrics

# HELP bar_metric Shows whether a bar has occurred in our cluster
# TYPE bar_metric counter
bar_metric 1
# HELP foo_metric Shows whether a foo has occurred in our cluster
# TYPE foo_metric counter
foo_metric 1
...
...
...
```

That's pretty much it. Really not that bad to implement from scratch, but of course and useful metrics will be more detailed than what we've done here. As far as rolling this out, you'd simply need to create a Docker image with your new Golang binary and deploy that image in a Kubernetes pod. Using the right annotations that we talked about earlier, your new metrics should be exposed automatically. Hit me up with any questions, but I won't swear to be a pro at Go or Prometheus :)