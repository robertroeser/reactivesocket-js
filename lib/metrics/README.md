# metrics

## Architecture

The library is composed of two central objects: the `Recorder` and the `Aggregator`.

You use the `Recorder` to create `Counters` and `Timers`, then the `Counters` and
`Timers` are used to send events to the `Recorder`.

On the other end, the `Aggregator` listen to events emitted by the `Counters` / `Timers`
and aggregate them. The default `Aggregator` aggregates counter events as a single
value (the latest) and timer event into an histogram.
But nothing is preventing you for having different aggregating strategy, for
instance you may want to keep a min/avg/latest/max structure for every counters.

The library provides two histograms implementations, one based on the classical
bucketed algorithm and one based on the more inovative frugal streaming
estimation. The first one is as precise as you want with an impact on speed and
memory footprint, the second one is extremly fast with almost no memory impact
but is less precise.

## Usage

Initializing the library simply like this:

```javascript
var recorder = new Recorder();
var aggregator = new Aggregator();
```

Creating/using counter/timer:

```javascript
var myCounter = recorder.counter('my_counter');
myCounter.incr();
myCounter.incr();

var latencyTimer = recorder.timer('latency');
var id = latencyTimer.start();
// do something...
latencyTimer.stop(id);
```

Creating a report:

```javascript
aggregator.report();
aggregator.clear();
```

This will generate a report like this:

```javascript
{
    counters: {
        my_counter: 2
    },
    histograms: {
        latency: {
            min: 1,
            max: 56,
            p50: 13.4,
            p90: 34.1,
            p99: 53.2
        }
    }
}
```

## Scoping

The `Recorder` has a `scope` method, which will return a new `Recorder`, every
new `Counter`/`Timer` created with this new recorder will have their name
prefixed by the scope value you provided during scoping.

Scoping is encouraged for categorizing metrics into different namespaces.

## Configuration

The configuration is pretty flexible and you can control how you send event at
both recording-time and aggregation-time.

In the following example, I disabled all timers execpt `request_latency_ms`, I
also swaped out the default histogram for a more precise one with more quantiles,
and also created a new metrics `connections/current` out of two existing
counters (`connections/add` and `connections/remove`).

```javascript
var config = {
    recorder: {
        timer: function (recorder, name, tags) {
            if (name === 'request_latency_ms') {
                return DefaultTimer(recorder, name, tags);
            } else {
                return NullTimer;
            }

        }
    },
    aggregator: {
        histogram: function (name) {
            return new BucketedHistogram({
                max: 60 * 1000,                    // 1 minute
                error: 2 / 100,                    // 2% precision
                quantiles: [0.5, 0.9, 0.99, 0.999] // default quantiles
            });
        },
        composites: function (counters, histograms) {
            var current = counters['connections/add'] - counters['connections/remove'];
            return ['connections/current', current];
        },
    }
};

var recorder = new Recorder(config.recorder);
var aggregator = new Aggregator(recorder, config.aggregator);
```

## Convention

It's always better to adopt similar convention in a project, here are the
conventions we picked:

### Stats names
Counters and Timers names are `snake_case` and should be postfixed by the unit
they represent, e.g `request_latency_ms`.

### Passing Recorder
Recorder has to be passed from Object to Object, the convention is that you
never scope the Recorder before passing it, you let the receiving class scope it
with its desired namespace.

e.g.

```javascript
var conn = new Connection(recorder)

// ...

function Connection(_recorder) {
    var recorder = _recorder.scope('connection');
    this.add = recorder.counter('add');
}
```