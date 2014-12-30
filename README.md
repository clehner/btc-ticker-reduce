# btc-ticker-reduce

BTC ticker with volume-weighted sliding average and line-based
output, written in Awk.

## Data source

`btc-ticker-reduce` uses data from Bitcoinchart's experimental
[streaming socket API](http://bitcoincharts.com/about/markets-api/#footer),
for which the following disclaimer is made: *"This service is strictly for
personal use. Do not assume this data to be 100% accurate or write trading
bots that rely on it."*

## Overview

The way it works is as follows. We listen to output from the TCP socket from
Bitcoincharts. Each line represents a trade that took place on an exchange.  We
filter for trades in the secondary currency of our choice. Then we take the
volume (in BTC) and amount (in the secondary currency) and add it to our
running totals. We discount the previous running total amount and volume by a
factor depending on the amount of time since our last observed trade.  The
discount factor exponentially increases as the time since last trade increases,
and depends on the `halflife` variable. `halflife` represents the number of
seconds since the last trade after which the running total and volume is
weighted by half of what it was weighted at the time of the last trade. We
output the running average after the weight adjustment, on its own line, one
line per trade.

Sometimes trades from LocalBitcoins trades are encountered which have
timestamps from far in the past, e.g. several days or weeks. It may be that
these timestamps were of when the trade was initiated, and they were only
finalized presently. We treat these trades as if they happened presently. When
finding such a trade exceeding an age threshold, we note the timestamp
difference since the trade before it. Then we wait until subsequent trades
"catch up" to the previous time by negating this time difference, before using
exponential backoff again. In this way we maintain the running total/volume
when processing far out-of-order trades which would otherwise overwhelm the
running total/volume with large exponent values.

## Usage

Use `netcat`/`ncat`/`nc` or `socat` or similar to pipe the socket data through
`btc-ticker-reduce`.

```
nc api.bitcoincharts.com 27007 | btc-ticker-reduce -v halflife=360 -v currency=USD
```

Alternatively, pipe the socket to a file and then read it back later.
```
nc api.bitcoincharts.com 27007 >> btc.json
...
^C
btc-ticker-reduce -v halflife=360 -v currency=USD btc.json
```

## Options/variables

* **halflife** (`-v halflife=3600`) - amount of smoothing to do, in seconds. Use
  30 (half a minute) for very little smoothing, or 3600 (1 hour) for lots of
  smoothing.

* **currency** (`-v currency=USD`) - which secondary currency to use. e.g. USD,
  CNY, EUR.

* **format** (`-v format=%f`) - printf format for the ticker price output,
  e.g. `$%.2f` for two decimal places and a dollar sign.

## Future work

* Keep track of moving averages per exchange, so that the outliers of e.g.
  LocalBitcoins can be better smoothed out.

* Get exchange rates between foreign currencies so that all trades can be
  used for data, rather than just trades in one foreign currency at a time.

* Handle possible changes to the JSON input, i.e. fields being in an unexpected
  order.

## Use cases

* Put the ticker into a status bar using e.g.
  [i3bar](http://i3wm.org/i3bar/) and [i3bar-mux](https://github.com/clehner/i3bar-mux).

## Why awk?

It is a simple enough language for a simple program. It is lightweight and
portable. It is good for processing streams of text.

## License

[![CC0](http://i.creativecommons.org/p/zero/1.0/88x31.png)](https://creativecommons.org/publicdomain/zero/1.0/)

To the extent possible under law, Charles Lehner has waived all copyright and
related or neighboring rights to this program. This work is published from the United States.
