# Analyzing the increase function of Prometheus

This article attempts to explain how the `increase` function of Prometheus works, and how to get an intuition about it when analyzing a plot. This is the first chapter of a 2-part series about using `increase` for anomaly detection.

When I first used `increase`, I thought that it was just a derivative for discrete functions. Which is true, but the hard part came when I had to interpret some Grafan plots that were using `increase`. My intuition about derivatives of continuous functions didn't really help me have a better understanding. This article will help you to get that missing intuition that you need when doing visual analysis on plots based on `increase`.

[The documentation](https://prometheus.io/docs/prometheus/latest/querying/functions/#increase) for the function states:
> increase(v range-vector) calculates the increase in the time series in the range vector. Breaks in monotonicity (such as counter resets due to target restarts) are automatically adjusted for. The increase is extrapolated to cover the full time range as specified in the range vector selector, so that it is possible to get a non-integer result even if a counter increases only by integer increments.

> increase should only be used with counters (for both floats and histograms). It is syntactic sugar for rate(v) multiplied by the number of seconds under the specified time range window, and should be used primarily for human readability. Use rate in recording rules so that increases are tracked consistently on a per-second basis.

When I first read that, I found it lacking in detail. I needed a mathematical definition of `increase`. So I built one myself, starting from the basics.

**Disclaimer:** I will not explain everything about the function, since some details are not that relevant. If you need an authoritative reference, [you can check the golang source code](https://github.com/prometheus/prometheus/blob/9c23509790a38e4f5ec38b0c60c91d2a4fb45bd0/promql/functions.go#L469-L472), which I found pretty easy to read (you need to know some high-school-level math, though). Some of my explanations are based on that source code.

## Range vector

One of the types commonly used for time series is a range vector:
```
v[range] = (t1, v1), (t2, v2), (t3, v3), ..., (tn, vn)
```
Which is a vector of tuples, where each tuple has a timestamp and a value. The timestamps are increasing. The range acts like a selector. It can be represented by an interval defined by `rangeStart` and `rangeEnd`, and selects all the values in the vector whose timestamps are in that interval: `rangeStart <= ti <= rangeEnd`. Prometheus encourages you to use a single value for ranges, and `rangeEnd` is assumed to be the current time. For example `v[5h]` is selecting an interval defined by `rangeEnd = now()` and `rangeStart = rangeEnd - 5h`.

## Instant vectors

Sometimes you might confuse instant vectors with range vectors. For example, the following is a range vector `counter[5h]`. But `increase(counter[5h])` is an instant vector, which means that it's one value.

However, when you enter these expressions in Grafana, if you give it a range vector, it will tell you:
> invalid expression type "range vector" for range query, must be Scalar or instant Vector

So, in order to plot something, you need to provide an instant vector to Grafana. But the result will actually be a set of points plotted within a range configured from the Grafana dashboard. That's because Grafana actually queries Prometheus for a range vector.

Wait, so is it that really an instant vector or not? The answer is: the expression you give to Grafana is an instant vector, but before sending the query to Prometheus, it actually selects a range for it (depending on how you configure the dashboard). So, if the dashboard has a plot for the last 3 hours, then the expression `increase(counter[5h])` turns into the query `increase(counter[5h])[3h]`. If you are confused why there are two ranges, you can think of it like a for loop: you query `increase(counter[(rangeEnd-5h, rangeEnd)])` for each step point over the interval of 3 hours by varying the value of `rangeEnd` from `now() - 3h` until `now()`.

It's not relevant for this article the exact query Grafana sends. I just want it to be clear that `increase(v[range])` returns exactly one value. If you want to plot it, you'll need to evaluate it multiple times by shifting the right side of the range across the x-axis of the time plot.

## Monotonicity

Let's first clear one of the easier issues when it comes to time series: **counter resets**.

> Breaks in monotonicity (such as counter resets due to target restarts) are automatically adjusted for

How is the adjustment done, exactly?

It means that at some point, you would have $v_{i} > v_{i+1}$. If you just select a range vector that contains this, the corresponding values will be plotted - nothing unexpected. However, when you need to apply some functions to these range vectors, Prometheus will make the following assumption: if a break in monotonicity happens, it means that the counter was reset to 0. In its calculations, it will add an offset every time this happens, to turn the series into a monotonic one.

This means that the new value will be $v_{i+1}' = v_{i} + v_{i+1}$. The offset is $v_{i}$, and it will be added to all the values that follow $v_{i}$. For example, if a drop to 0 happens (i.e. $v_{i+1}$ becomes 0), that sample will actually take the value $v_{i+1}' = v_{i}$, as if no change would've happened.

This only makes sense when our metric is actually a counter, like the documentation of `increase` states:
> increase should only be used with counters (for both floats and histograms)

Here's a visual example. The following is a counter with resets:
![Plot of counter that has 2 resets](img/counter_initial.png "A counter with resets")

And here it is how Prometheus will interpret it when fed to the `increase` function:
![Plot of the same counter without resets](img/counter_no_resets.png "The same counter without resets")

Here's a place in the implementation of the functions [where counter resets are handled](https://github.com/prometheus/prometheus/blob/9c23509790a38e4f5ec38b0c60c91d2a4fb45bd0/promql/functions.go#L243-L248):
```go
    for i, currPoint := range samples.Floats[1:] {
        prevPoint := samples.Floats[i]
        if currPoint.F < prevPoint.F || (i+1 < len(startTimestamps) && isStartTimestampReset(startTimestamps[i], prevPoint.T, startTimestamps[i+1], currPoint.T)) {
            resultFloat += prevPoint.F
        }
    }
```


## Extrapolation

Now, I will explain the following part:
> The increase is extrapolated to cover the full time range as specified in the range vector selector

Assuming `range := (rangeStart, rangeEnd)`, then the first selected sample in `v[range]` will be `(t1, v1)`, and `t1 >= rangeStart`. The range doesn't need to match the sampled timestamps exactly, so it is possible to have `t1 > rangeStart`.

As a consequence, instead of looking at `(t1, v1)`, Prometheus generates an extrapolated point `(t1', v1')`, which is extrapolated by assuming that the counter is linear. The points used for extrapolation are `(t1, v1)` and `(tn, vn)`. A similar thing happens for the right side of the interval - instead of using `(tn, vn)`, an extrapolated point `(tn', vn')` is used.

Usually, `t1' = rangeStart` and `tn' = rangeEnd`, but if these timestamps are too far from the actual ones, then Prometheus does some clipping. However, it is guaranteed that `t1' <= t1` and `tn' >= tn`.

Then `increase` function computes the difference `vn' - v1'` instead of `vn - v1`.

**Disclaimer:** The actual implementation of Prometheus handles some additional cases, and it doesn't explicitly generate `v1'` and `vn'`. It knows from the start that we want the difference between the last and the first sample, so it incorporates the extrapolation formula directly in the final result.

## The relation with rate

> It is syntactic sugar for rate(v) multiplied by the number of seconds under the specified time range window

[If we look at the documentation of `rate`](prometheus.io/docs/prometheus/latest/querying/functions/#rate), it is very similar to `increase`, so I will only paste here the interesting bit:
> rate(v range-vector) calculates the per-second average rate of increase of the time series in the range vector

*Average rate of increase* - if you're not familiar with TSDBs, this might sound ambiguous. Average relative to what? The number of samples? A time-weighted average? After reading [the source code for `extrapolatedRate`](https://github.com/prometheus/prometheus/blob/9c23509790a38e4f5ec38b0c60c91d2a4fb45bd0/promql/functions.go#L188), here's my own definition:

Given a range vector, `rate` computes the net increase divided by the total interval in seconds, which depends on the first and the last sample in the range:
```
rate(v[range]) = (vn - v1) / (tn - t1)
```

Going back to the definition of `increase`:
```
Simplified formula: increase(v[range]) = (vn - v1) / (tn - t1) * (rangeEnd - rangeStart)
```

But this is not the exact formula. It doesn't take into consideration the extrapolation. It also turns out that the *range window* is also clipped to the extrapolation.

In the next section, I'll wrap everything up.

## Formula

```
All the timestamps are measured in seconds.
Let range := (rangeStart, rangeEnd)
Let v[range] := [(v1, t1), (v2, t2), ..., (vn, tn)], where the timestamps are increasing and measured in seconds.
Assume the values are also increasing. Otherwise, adjust v[range] accordingly, as mentioned in the monotonicity section.
Let (v1', t1') and (vn', tn') be the extrapolated endpoints of v[range] (in practice, they are not too far away from (v1, t1) and (vn, tn)).
```

If we take extrapolation into consideration, the rate will be:
```
rate(v[range]) = (vn' - v1') / (tn' - t1')
```

What the documentation doesn't mention is that `(rangeStart, rangeEnd)` are replaced with `(t1', tn')`. If we apply the formula for `increase`, we get:
```
increase(v[range]) = rate(v[range]) * (tn' - t1') = vn' - v1'
```

Or an equivalent formula (based on the fact that the extrapolation is linear):
```
factor = (tn' - t1') / (tn - t1)
increase(v[range]) = (vn - v1) * factor
```

## Building the intuition

Even when you know the formula for `increase`, that's just one point. The real power of `increase` comes when it's sweeped across a range. I'll try to show you some examples and how to build an intuition about it.

Cheat sheet:
![Increase cheat sheet](img/increase_cheat_sheet.png)
