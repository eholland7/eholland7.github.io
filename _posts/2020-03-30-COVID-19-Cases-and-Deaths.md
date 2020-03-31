---
layout: post
title: COVID-19 Cases and Deaths
subtitle: An exploration into how a pandemic visually develops
tags: [COVID-19,d3,JavaScipt]
---

<style>

.counties {
  fill: #fff;
}

.states {
  fill: none;
  stroke: #fff;
  stroke-linejoin: round;
}

.bubbles {
  stroke: #000;
  fill-opacity: .35;
}

.bubbles-legend {
  stroke: #900;
  fill-opacity: 0;
}

.tip {
    position: absolute;
    padding: 5px;
    font: 12px sans-serif;
    color: white;
    background: dimgray;
    border: 0px;
    border-radius: 8px;
    pointer-events: none;
}
.label-text {
    font-size: 25px;
    font-family: sans-serif;
}

.title-text {
  font: 25px sans-serif;
  font-weight: bold;
  color: dimgray;
}

.subtitle-text {
  font: 15px sans-serif;
  font-weight: bold;
  color: dimgray;
}

.avg-text {
  font: 12px sans-serif;
  color: dimgray;
}

.btn-holder1 {
  position: absolute;
  top: 63%;
  left: 93%;
  transform: translate(-50%, -50%);
}

.btn-holder2 {
  position: absolute;
  top: 68%;
  left: 93%;
  transform: translate(-50%, -50%);
}


</style>

  <div class="col-md-6" id="chartarea">
    <div class="btn-holder1">
      <div id="buttons">
          <button id="bubblesOn">Display Cases</button>
      </div>
   </div>
   <div class="btn-holder2">
     <div id="buttons">
         <button id="bubblesOff">Hide Cases</button>
     </div>
  </div>
</div>

<script src="../lib/d3.v5.min.js"></script>
<script src="../lib/d3-scale-chromatic.v1.min.js"></script>
<script src="../lib/topojson.v2.min.js"></script>
<script src="../lib/d3-simple-slider.min.js"></script>
<script src="../lib/d3-tip.min.js"></script>
<script>

//create chart
var svg = d3.select("#chartarea").append("svg")
    .attr("width", 1000)
    .attr("height", 600);

var covid_cases = d3.map();
var covid_deaths = d3.map();
var regionMap = d3.map();
var countyMap = d3.map();
var parseTime = d3.timeParse("%Y-%m-%d");
var projection = d3.geoAlbersUsa().scale(1000).translate([390, 305])
var path = d3.geoPath().projection(projection);
var dates = [];

//legend -- for deaths
var x = d3.scaleLinear()
    .domain([1, 50])
    .rangeRound([325, 825]);

var rangeGreys = ["#ffffff","#f0f0f0","#eaeaea","#d9d9d9","#c5c5c5","#bdbdbd", "#a9a9a9"
      ,"#a0a0a0","#969696","#888888","#828282","#737373","#646464","#525252","#3e3e3e"
      ,"#252525","#000000"];

var color = d3.scaleThreshold()
    .domain(d3.range(0, 50, 3))
    .range(rangeGreys);

var g = svg.append("g")
    .attr("class", "key")
    .attr("transform", "translate(-150,45)");

g.selectAll("rect")
  .data(color.range().map(function(d) {
      d = color.invertExtent(d);
      if (d[0] == null) d[0] = x.domain()[0];
      if (d[1] == null) d[1] = x.domain()[1];
      return d;
    }))
  .enter().append("rect")
    .attr("height", 8)
    .attr("x", function(d) { return x(d[0]); })
    .attr("width", function(d) { return x(d[1]) - x(d[0]); })
    .attr("fill", function(d) { return color(d[0]); });

g.append("text")
    .attr("class", "caption")
    .attr("x", x.range()[0] + 175)
    .attr("y", -6)
    .attr("fill", "#000")
    .attr("text-anchor", "start")
    .attr("font-weight", "bold")
    .text("Deaths from COVID-19");

g.call(d3.axisBottom(x)
    .tickSize(13)
    .tickFormat(function(x, i) { return (i === 16) ? x + "+" : x ; })
    .tickValues(color.domain()))
  .select(".domain")
    .remove();

//legend -- for cases
var bubbles_legend = svg.selectAll(".bubbles-legend")
    .data([2000, 4000, 8000])
    .enter().append("circle")
    .attr("class", "bubbles-legend")
    .attr("r", function(d) {
      return Math.sqrt(d) / (Math.PI / 1.5);
    })
    .attr("transform", "translate(930,470)")
    .attr("cy", d => -(Math.sqrt(d) / (Math.PI/1.5)));

svg.append("text")
      .attr("transform", "translate(930,440)")
      .attr("text-anchor", "middle")
      .style("font", "10px sans-serif")
      .attr("fill", "#900")
      .text("2k");
svg.append("text")
      .attr("transform", "translate(930,420)")
      .attr("text-anchor", "middle")
      .style("font", "10px sans-serif")
      .attr("fill", "#900")
      .text("4k");
svg.append("text")
      .attr("transform", "translate(930,395)")
      .attr("text-anchor", "middle")
      .style("font", "10px sans-serif")
      .attr("fill", "#900")
      .text("8k");
svg.append("text")
      .attr("transform", "translate(930,490)")
      .attr("text-anchor", "middle")
      .style("font", "10px sans-serif")
      .text("Cases of COVID-19");

// tooltip
function strong(text) {
    return "<strong>" + text + "</strong>"
}

var tip = d3.tip()
    .attr("class", "tip")
    .offset([-5, 0])
    .html(function(d) {
        var county = d.properties.name;
        var state = regionMap.get(+d.id);
        var cases = covid_cases.get(d.id);
        var deaths = covid_deaths.get(d.id);
        if (cases === undefined) {
             cases = 0;
        }
        if (deaths === undefined) {
             deaths = 0;
        }
        return "State: " + strong(state) +
            "<br>County: " + strong(county) +
            "<br>Cases: " + strong(cases) +
            "<br>Deaths: " + strong(deaths) + "</span>";
    });

svg.call(tip);

function handleMouseOver(d) {
    tip.show(d);
    d3.select(this)
        .attr("stroke", "white")
    };
function handleMouseOut(d) {
    tip.hide(d);
    d3.select(this)
        .attr("stroke", "none")
        .attr("stroke", "none")
    };

//slider to view the data as it happened
//set og value
updateData(parseTime('2020-03-28'));

var dispatch = d3.dispatch("input", "statechange");
var slider = d3.sliderRight()
    .min(parseTime("2020-01-21"))
    .max(parseTime("2020-03-28"))
    .height(350)
    .tickFormat(d3.timeFormat("%m-%d"))
    .tickValues(dates.forEach(element => parseTime(element)))
    .default(parseTime("2020-03-28"))
    .on("end", function(value) {

      updateData(value);
      d3.selectAll(".bubbles").remove();
      d3.selectAll(".bubbles")
        .transition()
        .duration(10)
        .attr("r", function(d) {
          d.initCovid = covid_cases.get(d.id);
          if (d.initCovid === undefined || d.initCovid < 1) {
              d.value = 0;
          } else {
              d.value = d.initCovid;
          }
          return Math.sqrt(d.value) / (Math.PI/1.5);
        })
        .attr("cx", function(d) { return path.centroid(d)[0] })
        .attr("cy", function(d) { return path.centroid(d)[1] })
        .style("fill", "#900");
    });

svg.append("g")
    .call(slider)
    .attr("transform", "translate(770,160)");

svg.append("text")
    .attr("text-anchor", "start")
    .attr("y", 170)
    .attr("x", 850)
    .attr("class", "subtitle-text")
    .text("Date Selection Info:");

svg.append("text")
    .attr("text-anchor", "start")
    .attr("y", 200)
    .attr("x", 850)
    .attr("class", "avg-text")
    .text("Use the slider on the left to");
svg.append("text")
    .attr("text-anchor", "start")
    .attr("y", 215)
    .attr("x", 850)
    .attr("class", "avg-text")
    .text("display deaths and cases ");
svg.append("text")
    .attr("text-anchor", "start")
    .attr("y", 230)
    .attr("x", 850)
    .attr("class", "avg-text")
    .text("for a date between January");
svg.append("text")
    .attr("text-anchor", "start")
    .attr("y", 245)
    .attr("x", 850)
    .attr("class", "avg-text")
    .text("21st and March 25th. ");
svg.append("text")
    .attr("text-anchor", "start")
    .attr("y", 265)
    .attr("x", 850)
    .attr("class", "avg-text")
    .text("Toggle the buttons below to ");
svg.append("text")
    .attr("text-anchor", "start")
    .attr("y", 280)
    .attr("x", 850)
    .attr("class", "avg-text")
    .text("get rid of the bubbles.");

//button details
d3.selectAll("button")
  .on("click", function() {
    var bubbleType = this.id;
    d3.selectAll(".bubbles")
        .transition()
        .duration(500)
        .attr("r", function(d) {
          return (bubbleType == "bubblesOff" ? 0 : Math.sqrt(covid_cases.get(d.id)) / (Math.PI/1.5));
        })
  });

/*****************************************************************************/

//get data
function updateData(newDate) {

  covid_cases.clear();
  covid_deaths.clear();

  var promises = [
    d3.json("../counties-10m.json"),
    d3.csv("../data/covid-counties.csv", function(data) {
      if (parseTime(data.date) <= newDate) {
        dates.push(parseTime(data.date));
        covid_cases.set(data.fips, +data.cases);
        covid_deaths.set(data.fips, +data.deaths);
        countyMap.set(data.fips, data.county);
      }
    }),
    d3.csv("../data/state_county_map.csv", function(d) {
      regionMap.set(+d.fips, d.state);
    })
  ]

  Promise.all(promises).then(ready)
};

function removeDuplicates(array) {
  let x = {};
  array.forEach(function(i) {
    if(!x[i]) {
      x[i] = true
    }
  })
  return Object.keys(x)
};

//plot data
function ready([us]) {

  dates = removeDuplicates(dates);
  dates.sort(function (a, b) {
    var key1 = new Date(a.date);
    var key2 = new Date(b.date);

    if (key1 < key2) {
        return 1;
    } else if (key1 == key2) {
        return 0;
    } else {
        return -1;
    }
  });

  svg.append("g")
      .attr("class", "counties")
    .selectAll("path")
    .data(topojson.feature(us, us.objects.counties).features)
    .enter().append("path")
      .attr("fill", "#DCDCDC")
      .attr("fill", function(d) {
          d.initCovidd = covid_deaths.get(d.id);
          if (d.initCovidd === undefined || d.initCovidd < 1) {
            d.value = 0;
          } else {
            d.value = d.initCovidd;
          }
          return color(d.value);
        })
      .attr("d", path)
      .on("mouseover", handleMouseOver)
      .on("mouseout", handleMouseOut);

  svg.append("path")
      .datum(topojson.mesh(us, us.objects.states, function(a, b) { return a !== b; }))
      .attr("class", "states")
      .attr("d", path);

  var bubbies = svg.selectAll(".bubbles")
      .data(topojson.feature(us, us.objects.counties).features)
      .enter().append("circle")
      .attr("class", "bubbles")
      .attr("r", function(d) {
          d.initCovid = covid_cases.get(d.id);
          if (d.initCovid === undefined || d.initCovid < 1) {
              d.value = 0;
          } else {
              d.value = d.initCovid;
          }
          return Math.sqrt(d.value) / (Math.PI/1.5);
      })
      .attr("cx", function(d) { return path.centroid(d)[0] })
      .attr("cy", function(d) { return path.centroid(d)[1] })
      .style("fill", "#900")
      .on("mouseover", handleMouseOver)
      .on("mouseout", handleMouseOut);
}
</script>
