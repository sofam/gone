# gone metrics

Fast Golang metrics library

Package gone/metric is an expandable library for metrics.
Initally only with a sink for sending data to statsd

The design goals:

* Basic statsd metric types (gauge, counter, timer)
* Client side buffered if needed.
* Fast

Timer and Histogram is basically the same except for the argument type.

Counter is reset to zero on each flush. Gauges are not.

## Permanent and ad-hoc meters

The library provides to different APIs for generating metrics events.

If you need Permanent meters (gauges/counters/timers...) which will be updated often with lots of data and want to use the lock free client side buffering to make it really fast, you can create explicit gauge/counter/timer/histogram objects using the client.NewXXX() methods.

If you, on the other hand, rarely record a metric event more than once, but have a lot of different named events, you can bypass the buffering datastructures and send the metric event directly to the sink by using client.Gauge(), client.Counter(), client.Timer()...

Such direct recording of metrics events will still (depending on sink implementation) be bufferend in the protocol layer - unless you request an explicit flush.

This is maybe illustrated best with a Counter example:

If you do:

```go
{
	counter := client.NewCounter("name")
	counter.Inc(1)
	counter.Inc(1)
}
```

Then buffering will only send an increase of "2" to the sink - and only put it on the wire when the sink flushes.

On the other hand, if you do:

```go
{
	client.Counter("name",1, false)
	client.Counter("name",1, true)
}
```

Then both metric events will be sent to the sink and the latter will also ask the sink to flush data to the wire immediately.

## Example

```go
import (
	"github.com/One-com/gone/metric"
	"github.com/One-com/gone/metric/sink/statsd"
	"log"
	"time"
)

	var flushPeriod = := metric.FlushInterval(4*time.Second)

	sink, err := statsd.New(
		statsd.Buffer(512),
		statsd.Peer("statsdhost:8125"),
		statsd.Prefix("prefix"))
	if err != nil {
		log.Fatal(err)
	}

	c := metric.NewClient(sink,flushPeriod)
	
	gauge   := c.NewGauge("gauge",flushPeriod)
	timer   := c.NewTimer("timer")
	histo   := c.NewHistogram("histo",flushPeriod)
	counter := c.NewCounter("counter",flushPeriod)

	g := 100
	for g != 0 {
		counter.Inc(1)
		gauge.Set(uint64(g))
		timer.Sample(time.Duration(g)*time.Millisecond)
		histo.Sample(int64(g))
		
		time.Sleep(time.Second)
		g--
	}
	c.Stop()	
}


```
