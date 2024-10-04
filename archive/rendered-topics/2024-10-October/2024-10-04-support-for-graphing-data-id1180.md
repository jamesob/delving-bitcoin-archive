# Support for graphing data

ajtowns | 2024-10-04 06:45:12 UTC | #1

One thing I've been wishing for is support for graphing data in a traditional x-y plot with just text markup and not having to generate a png externally or similar. Eventually I gave up and wrote my own [discourse component](https://github.com/ajtowns/discourse-plotly-theme-component) based on [plotly.js](https://github.com/plotly/plotly.js/), which I've now enabled here.

Here's an example graph based on a graph from [the cluster mempool proposal overview](https://delvingbitcoin.org/t/an-overview-of-the-cluster-mempool-proposal/393):

```plotly
data:
  - x: [0,300,400]
    y: [0,950,1050]
    mode: lines+markers
    name: Old mempool
  - x: [0,200,400]
    y: [0,700, 1150]
    name: New mempool
layout:
  title: Feerate Diagram
```

You can have a look at the source used to generate that graph via the "Raw Post" link at the bottom of this post, hidden behind the "...". One thing I like about it is that you can hover over the points on the graph to see their actual values; but you can also zoom in and pan around which might also be useful.

You can also do log graphs easily, just by adding `yaxis: {type: log, autorange: true}` to the `layout` section:

```plotly
data:
  - x: [0,300,400]
    y: [1,950,1050]
    mode: lines+markers
    name: Old mempool
  - x: [0,200,400]
    y: [1,700, 1150]
    name: New mempool
layout:
  title: Feerate Diagram
  yaxis: { type: log, autorange: true }
```

(Log axes work remarkably poorly if you have datapoints with a value of zero or less, of course...)

Plotly supports a lot of different graph types, eg drawing great circles on a world map:

```plotly
data:
  - type: scattergeo    
    lat: [ 40.7127, 51.5072 ]
    lon: [ -74.0059, 0.1275 ]
    mode: lines
    line:
      width: 4
      color: black
layout:
  title: London to NYC Great Circle
  showlegend: false
  geo:
      resolution: 50
      showland: true
      showlakes: true
      showocean: true
      landcolor: tan
      lakecolor: lightblue
      oceancolor: lightblue
      projection:
        type: equirectangular
      coastlinewidth: 2
      lataxis:
        range: [ 20, 60 ]
        showgrid: true
        tickmode: linear
        dtick: 10
      lonaxis:
        range: [-100, 20]
        showgrid: true
        tickmode: linear
        dtick: 20
```

Not sure how useful any of that is likely to be, but it's more effort to disable it than to leave it in.

You can get a general idea of what's available from https://plotly.com/javascript/ and in particular, the code in a `plotly` code block is just treated as a yaml with the `data` and `layout` sections passed into plotly as the `data` and `layout` arguments of plotly's `newPlot()`.

-------------------------

