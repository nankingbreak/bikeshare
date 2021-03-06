---
title: Bike Sharing Systems
subtitle: rCharts + Shiny
author: Ramnath Vaidyanathan
github: {user: ramnathv, repo: bikeshare, branch: "gh-pages"}
framework: minimal
mode: selfcontained
highlighter: prettify
hitheme: twitter-bootstrap
assets:
  css: "http://fonts.googleapis.com/css?family=PT+Sans"
---

<style>
 /* body{background: white;} */
 ol.linenums{margin-left: -8px;}
 p, li{text-align: justify;font-size: 15px;line-height:1.5em;font-family: "PT Sans"}
</style>

## Visualizing Bike Sharing Networks

<!-- AddThis Button BEGIN -->
<div class="addthis_toolbox addthis_default_style ">
<a class="addthis_button_facebook_like" fb:like:layout="button_count"></a>
<a class="addthis_button_tweet"></a>
<a class="addthis_button_pinterest_pinit"></a>
<a class="addthis_counter addthis_pill_style"></a>
</div>
<script type="text/javascript">
  var addthis_config = {"data_track_addressbar":false};
</script>
<script type="text/javascript" src="//s7.addthis.com/js/300/addthis_widget.js#pubid=ra-4fdfcfd4773d48d3"></script>
<!-- AddThis Button END -->

A couple of months ago I had posted an interesting application of using rCharts and Shiny to visualize the CitiBike system in NYC. I had always wanted to write a tutorial about its inner workings, so that it would be useful to others looking to build similar visualizations, and I finally got around to doing it. Along the way, I managed to extend the visualization to around 100 bike sharing systems across the world. The final application can be viewed [here](http://glimmer.rstudio.com/ramnathv/BikeShare). 

<a href="http://glimmer.rstudio.com/ramnathv/BikeShare">
<img src=http://www.clipular.com/c?10951071=aD5PWoWf3MjZaDGbvSxV7ZyIeM4&f=.png>
</img>
</a>

If you are impatient, you can view all the code on my [github repo](http://github.com/ramnathv/bikeshare) and run the application directly from github.


```r
require(shiny)
runGitHub("bikeshare", "ramnathv", ref = "gh-pages", subdir = "app")
```







If you want a more detailed explanation of how the app was built, read on.

### Introduction

My mantra for building interactive visualizations involves three steps, and it has worked well for me most of the time.

1. Get Data.
2. Create Visualization.
3. Wrap in Shiny/AngularJS!

Let me expand on this and build the web app one step at a time.

### Get Data

The first step is to get the data on availability of bikes in a city. Thankfully, the folks at [http://api.citybik.es/](CityBikes) have provided an API that allows one to programatically retrieve the availabilities across more than 100 bike sharing networks across the world. I like to wrap my analysis workflow into small functions, so that it is modular. There are two things that my `getData` function does.

1. Fetch data for a given network using `httr`. (thanks @hadley)
2. Add `fillColor` and `popup` to each station of the network.


```r
getData <- function(network = 'citibikenyc'){
  require(httr)
  url = sprintf('http://api.citybik.es/%s.json', network)
  bike = fromJSON(content(GET(url)))
  lapply(bike, function(station){within(station, { 
   fillColor = cut(
     as.numeric(bikes)/(as.numeric(bikes) + as.numeric(free)), 
     breaks = c(0, 0.20, 0.40, 0.60, 0.80, 1), 
     labels = brewer.pal(5, 'RdYlGn'),
     include.lowest = TRUE
   ) 
   popup = iconv(whisker::whisker.render(
      '<b>{{name}}</b><br>
        <b>Free Docks: </b> {{free}} <br>
         <b>Available Bikes:</b> {{bikes}}
        <p>Retreived At: {{timestamp}}</p>'
   ), from = 'latin1', to = 'UTF-8')
   latitude = as.numeric(lat)/10^6
   longitude = as.numeric(lng)/10^6
   lat <- lng <- NULL})
  })
}
```


Now that we have the data, it is time to visualize it.

### Create Visualization

Given the nature of the data, it is best to visualize on a map. [rCharts](http://rcharts.io) provides bindings to the [Leaflet](leafletjs.com) library, which makes mapping really easy. The `plotMap` function essentially does the following:

1. Creates a new instances of a Leaflet map.
2. Sets the map's provider, width, height, center and zoom level.
3. Adds the network data retrieved as a geoJSON layer.
4. Configures the properties of each point and popup to display on click.


```r
plotMap <- function(network = 'citibikenyc', width = 1600, height = 800){
  data_ <- getData(network)
  center_ <- getCenter(network, networks)
  L1 <- Leaflet$new()
  L1$tileLayer(provider = 'Stamen.TonerLite')
  L1$set(width = width, height = height)
  L1$setView(c(center_$lat, center_$lng), 13)
  L1$geoJson(toGeoJSON(data_), 
    onEachFeature = '#! function(feature, layer){
      layer.bindPopup(feature.properties.popup)
    } !#',
    pointToLayer =  "#! function(feature, latlng){
      return L.circleMarker(latlng, {
        radius: 4,
        fillColor: feature.properties.fillColor || 'red',    
        color: '#000',
        weight: 1,
        fillOpacity: 0.8
      })
    } !#"
  )
  L1$enablePopover(TRUE)
  L1$fullScreen(TRUE)
  return(L1)
}
```


We can test this function by plotting the availabilities of bikes in NYC. You can play with `plotMap` and change the default color palette, or popup details, and see how it affects the map.


```r
plotMap('citibikenyc', 600, 300)
```


<iframe src='assets/img/citibikenyc.html' width = 600 frameBorder="0"></iframe>

Now that we have successfully visualized the bike sharing system for NYC, we can get to the exciting task of wrapping this up in a Shiny application, where the user can interactively choose the bike sharing system, whose availabilities they want to visualize. Before, we can do that, we need the names of these systems to passed to `plotMap`. Thankfully, the [http://api.citybik.es/](CityBikes) API provides easy access to this. The `getNetworks` function retrieves this data.


```r
getNetworks <- function(){
  require(httr)
  if (!file.exists('networks.json')){
    url <- 'http://api.citybik.es/networks.json'
    dat <- content(GET(url))
    writeLines(dat, 'networks.json')
  }
  networks <- RJSONIO::fromJSON('networks.json')
  nms <- lapply(networks, '[[', 'name')
  names(networks) <- nms
  return(networks)
}
```


### Wrap in Shiny

This is the easiest part of the whole tutorial. Shiny requires two files `ui.R` and `server.R`, that contain the UI and server logic respectively.

For the UI, I make use of a basic bootstrap page that ships with Shiny. Lines 5 - 7 add links to a custom style file and javascript file that allow me to add a collapsible __credits__ box at the bottom left of the page. I use a `selectInput` for users to select the network they want to visualize, and populate it with an alphabetically sorted list of names of all the networks, initialized to `citibikenyc`. Finally, I use the `mapOutput` function which adds a div containter named `map_container` that houses the map.



```r
require(shiny)
require(rCharts)
networks <- getNetworks()
shinyUI(bootstrapPage( 
  tags$link(href='style.css', rel='stylesheet'),
  tags$script(src='app.js'),
  includeHTML('www/credits.html'),
  selectInput('network', '', sort(names(networks)), 'citibikenyc'),
  mapOutput('map_container')
))
```


The server side code is even simpler than the UI and merely wraps the `plotMap` call inside `renderMap`, and passes the name of the network chosen by the user, `input$network` in place of the hard-coded `citibikenyc`.



```r
require(shiny)
require(rCharts)
shinyServer(function(input, output, session){
  output$map_container <- renderMap({
    plotMap(input$network)
  })
})
```



### Acknowledgements

1. [Vladimir Agafonkin](http://leafletjs.com) for Leaflet.
2. [CitiBikes](http://citybik.es/) for easy access to data.
3. [Joe Cheng](http://github.com/jcheng5) and RStudio for Shiny.
4. [Kenton Russell](http://github.com/timelyportfolio) and [Thomas Reinholdsson](http://github.com/reinholdsson) for some awesome work on rCharts.
5. [Yihui Xie](http://github.com/yihui) for knitr.
6. [Hadley Wickham](http://github.com/yihui) for httr and several other packages.




