---
layout: post
comments: true
author: Spencer
title: A Noob's Guide to Custom Prometheus Exporters (Revamped!)
tags: kubernetes golang prometheus
---

Someone was kind enough to send me an email thanking me for the [previous post](https://rsmitty.github.io/Prometheus-Exporters/) I created detailing how to create prometheus exporters using golang. While it was great to receive a note, I kind of panicked because I realized that I hadn't updated that post to reflect a much easier way of creating exporters that I had learned about. This post will hopefully shed some light on the better way. It may still be beneficial to read the previous post if you want some extra context around my initial thoughts on this.

## **Straight to the Code** ##

No messing around in this post. Creating a basic go program with a `/metrics` endpoint is pretty straight forward and, in fact, the initial `main.go` file is unchanged from the previous post.

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

Now, let's build our metrics. We'll do this by creating a collector.go file and adding a simple init function to register and seed our metrics with values.

- Create a new file called `collector.go` in the same directory.
- In the file, paste the following code.

```go
package main

import (
	"github.com/prometheus/client_golang/prometheus"
)

//Define the metrics we wish to expose
var fooMetric = prometheus.NewGauge(prometheus.GaugeOpts{
	Name: "foo_metric", Help: "Shows whether a foo has occurred in our cluster"})

var barMetric = prometheus.NewGauge(prometheus.GaugeOpts{
	Name: "bar_metric", Help: "Shows whether a bar has occurred in our cluster"})

func init() {
	//Register metrics with prometheus
	prometheus.MustRegister(fooMetric)
	prometheus.MustRegister(barMetric)

	//Set fooMetric to 1
	fooMetric.Set(0)

	//Set barMetric to 0
	barMetric.Set(1)
}
```

Once we've done that we're actually already pretty much finished! Note that the init function is a bit of go magic. It's run automatically before running the main function in your program. In the init function above, we simply register our metrics with the prometheus client and give them an initial value.

 - Hit the `/metrics` endpoint after starting your webserver with `go run main.go collector.go`.

```bash
$ curl 127.0.0.1:8080/metrics

# HELP bar_metric Shows whether a bar has occurred in our cluster
# TYPE bar_metric gauge
bar_metric 1
# HELP foo_metric Shows whether a foo has occurred in our cluster
# TYPE foo_metric gauge
foo_metric 0
...
...
...
```

Much quicker! From here, you would likely want to add some functions to your program that updates your metrics based on things occurring during runtime or based on a timer. We won't cover those here but as you can see, exposing metrics in this way is much easier than implementing the interfaces and whatnot that I was doing in the previous post. Hope this helps folks out!