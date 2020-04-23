---
layout: post
title: COVID-19 Predicted Cases in New England
subtitle: An investigation into the potential progression of COVID-19
tags: [COVID-19,D3,JavaScipt]
---


- Use the slider below to display deaths and cases for a date beginning January 21st.
- Toggle the buttons to display and hide the number of cases.
- Move your cursor over the map to view more detail on the exact number of cases and deaths.<br/>


<meta charset="utf-8">

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
  stroke-width: 0.25;
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

.btn-holder {
  position: absolute;
  top: 17%;
  left: 80%;
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
  <select>
      <option id="o1">Infection rate: 0.001; Recovery rate: 0.06</option>
      <option id="o2">Infection rate: 0.010; Recovery rate: 0.80</option>
      <option id="o3">Infection rate: 0.003; Recovery rate: 0.08</option>
  </select>
</div>

<script src="../lib/d3.v5.min.js"></script>
<script src="../lib/d3-scale-chromatic.v1.min.js"></script>
<script src="../lib/topojson.v2.min.js"></script>
<script src="../lib/d3-simple-slider.min.js"></script>
<script src="../lib/d3-tip.min.js"></script>
<script>

//create chart
var svg = d3.select("#chartarea").append("svg")
    .attr("width", 1230)
    .attr("height", 800);

var covid_cases = d3.map();
var covid_deaths = d3.map();
var regionMap = d3.map();
var countyMap = d3.map();
var parseTime = d3.timeParse("%Y-%m-%d");

var projection = d3.geoAlbersUsa().scale(4000).translate([-875, 900])
var path = d3.geoPath().projection(projection);
var dates = [];
var at_date;
var min_date;
var max_date;

var stats, counties, datelist, fipslist;
//legend -- for population
var x = d3.scaleLinear()
    .domain([1, 325000])
    .rangeRound([325, 875]);

var rangeGreys = ["#ffffff","#f0f0f0","#eaeaea","#d9d9d9","#c5c5c5","#bdbdbd", "#a9a9a9"
      ,"#a0a0a0","#969696","#888888","#828282","#737373","#646464","#525252","#3e3e3e"
      ,"#252525","#000000"];

var color = d3.scaleThreshold()
    .domain(d3.range(0, 325000, 20000))
    .range(rangeGreys);

var g = svg.append("g")
    .attr("class", "key")
    .attr("transform", "translate(60,-280)");//-150,45

g.selectAll("rect")
  .data(color.range().map(function(d) {
      d = color.invertExtent(d);
      if (d[0] == null) d[0] = x.domain()[0];
      if (d[1] == null) d[1] = x.domain()[1];
      return d;
    }))
  .enter().append("rect")
    .attr("width", 8)
    .attr("y", function(d) { return x(d[0]); })
    .attr("height", function(d) { return x(d[1]) - x(d[0]); })
    .attr("fill", function(d) { return color(d[0]); });

g.append("text")
    .attr("class", "caption")
    .attr("x", x.range()[0] + 95)
    .attr("y", -10)
    .attr("fill", "#000")
    .attr("transform", "rotate(90)")
    .style("font", "10px sans-serif")
    .text("Population by County");

g.call(d3.axisLeft(x)
    .tickSize(13)
    .tickFormat(function(x, i) { return (i === 16) ? x + "+" : x ; })
    .tickValues(color.domain()))
  .select(".domain")
    .remove();

//legend -- for cases
var bubbles_legend = svg.selectAll(".bubbles-legend")
    .data([20000, 50000, 100000])
    .enter().append("circle")
    .attr("class", "bubbles-legend")
    .attr("r", function(d) {
      return Math.sqrt(d) / (Math.PI * 2);
    })
    .attr("transform", "translate(700,550)")
    .attr("cy", d => -(Math.sqrt(d) / (Math.PI * 2)));

svg.append("text")
      .attr("transform", "translate(700,516)")
      .attr("text-anchor", "middle")
      .style("font", "10px sans-serif")
      .attr("fill", "#900")
      .text("20k");
svg.append("text")
      .attr("transform", "translate(700,490)")
      .attr("text-anchor", "middle")
      .style("font", "10px sans-serif")
      .attr("fill", "#900")
      .text("50k");
svg.append("text")
      .attr("transform", "translate(700,460)")
      .attr("text-anchor", "middle")
      .style("font", "10px sans-serif")
      .attr("fill", "#900")
      .text("100k");
svg.append("text")
      .attr("transform", "translate(700,563)")
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
        var county;
        if (d.id === '36047') {
          county = 'New York City';
        } else if (d.id === "29047") {
          county = 'Clay / Kansas City';
        } else if (d.id === '17031') {
          county = 'Cook / Chicago';
        } else {
          county = d.properties.name;
        }
        var state = regionMap.get(+d.id);
        var cases = covid_cases.get(at_rate).get(at_date).get(d.id);
        var pop = covid_deaths.get(at_rate).get(at_date).get(d.id);
        if (cases === undefined) {
             cases = 0;
        }
        return "State: " + strong(state) +
            "<br>County: " + strong(county) +
            "<br>Population: " + strong(pop) +
            "<br>Cases: " + strong(cases) +
            "</span>";
    });

svg.call(tip);

function handleMouseOver(d) {
    tip.show(d);
    d3.select(this)
        .attr("stroke", "white")
        .attr("stroke-width", "6")
    };
function handleMouseOut(d) {
    tip.hide(d);
    d3.select(this)
        .attr("stroke", "none")
    };

function checker(x) {
  return (x === undefined ? 0 : x);
}

function rad(id) {
  return ( (!show_bubbles) ? 0 : Math.sqrt(checker(covid_cases.get(at_rate).get(at_date).get(id))) / (Math.PI * 2));
}

//button details
var show_bubbles = true;
d3.selectAll("button")
  .on("click", function() {
    show_bubbles = (this.id == 'bubblesOn');
    redraw();
  });


//dropdown details
var show_og = 0;
d3.selectAll("select")
  .on("change", function(d) {
    console.log(d);
    console.log(this.selectedIndex);
    show_og = this.selectedIndex;
    at_rate = ratelist[show_og]
    redraw();
  });

/*****************************************************************************/

  var promises = [
    d3.json("../counties-10m.json"),
    // d3.csv("https://raw.githubusercontent.com/nytimes/covid-19-data/master/us-counties.csv", function(d) {
    d3.csv("../data/total_combined.csv", function(d) {
        return {date: d.date
                , county:d.county
                , state:d.state
                , fips:d.fips
                , cases: +d.cases * 1000
                , rates: d.rate
                , pop: +d.population};
    }),
    d3.csv("../data/state_county_map.csv", function(d) {
      regionMap.set(+d.fips, d.state);
    })
  ]

  Promise.all(promises).then(ready);

//preliminary plot
function ready(data) {

  us = data[0];
  stats = data[1];
  counties = data[2];
  counties.map(function (d) {
      regionMap.set(d['fips'], d['state']);
      countyMap.set(d['fips'], d['county']);
  });

  stats.forEach(function(item) {
    if (item['fips'] === "") {
      //its a random City
      if (item['county'] === "New York City") {
        //36047: KINGS COUNTY, FOR NYC
        item['fips'] = "36047";
      } else if (item['county'] === "Kansas City") {
        //29047: CLAY COUNTY, MO, FOR KANSAS CITY
        item['fips'] = "29047";
      }
    }
  });

  //get list of unique dates and fips
  datelist = [...new Set(stats.map(d=>d.date))];
  ratelist = [...new Set(stats.map(d=>d.rates))]
  fipslist = [...new Set(stats.map(d=>d.fips))];

  //make full list of dates, county values & set default to 0
  for (z=0; z<ratelist.length; z++) {
    r = ratelist[z];
    covid_cases.set(r, d3.map());
    covid_deaths.set(r, d3.map());
    for (i=0; i<datelist.length; i++) {
        d = datelist[i]; // the date
        covid_cases.get(r).set(d, d3.map());
        covid_deaths.get(r).set(d, d3.map());

        // covid_cases.set(d, d3.map());
        // covid_deaths.set(d, d3.map());
        for (j=0; j<fipslist.length; j++) {
            fips = fipslist[j];
            covid_cases.get(r).get(d).set(fips, 0);
            covid_deaths.get(r).get(d).set(fips, 0);
        }
    }
  }

  //assign data to right date
  for (i=0; i<stats.length; i++) {
      r = stats[i]; // the row
      covid_cases.get(r['rates']).get(r['date']).set( r['fips'], r['cases']);
      covid_deaths.get(r['rates']).get(r['date']).set(r['fips'], r['pop']);
  }

  at_date = datelist[datelist.length - 1];
  at_rate = ratelist[0];
  console.log(at_rate);
  console.log(ratelist[1]);
  min_date = datelist[0];
  max_date = datelist[datelist.length - 1];

  //slider to view the data as it happened
  var dispatch = d3.dispatch("input", "statechange");
  var slider = d3.sliderBottom()
      .min(parseTime(min_date))
      .max(parseTime(max_date))
      .width(500)
      .tickFormat(d3.timeFormat("%m-%d"))
      .tickValues(dates.forEach(element => parseTime(element)))
      .default(parseTime(max_date))
      .handle(
        d3
          .symbol()
          .type(d3.symbolCircle)
          .size(200)()
      )
      .on("end", function(value) {
        at_date = d3.timeFormat("%Y-%m-%d")(value);
        redraw();
      });

  svg.append("g")
      .call(slider)
      .attr("transform", "translate(220,20)");

  init_graph();
  redraw();
}

function init_graph() {
  svg.append("g")
      .attr("class", "counties")
    .selectAll("path")
    .data(topojson.feature(us, us.objects.counties).features.filter(function(d) {
      //23 = maine 50 = vermont
      return d.id.startsWith("23") || d.id.startsWith("50") || d.id.startsWith("44") || d.id.startsWith("33") || d.id.startsWith("09");
    }))
    .enter().append("path")
      .attr("fill", function(d) { return color(0); })
      .attr("d", path)
      .on("mouseover", handleMouseOver)
      .on("mouseout", handleMouseOut);

  svg.append("path")
      .datum(topojson.mesh(us, us.objects.counties, function(a, b) { return a !== b; }))
      .attr("class", "states")
      .attr("d", path);

  var bubbies = svg.selectAll(".bubbles")
      .data(topojson.feature(us, us.objects.counties).features.filter(function(d) {
        return d.id.startsWith("23") || d.id.startsWith("50") || d.id.startsWith("44") || d.id.startsWith("33") || d.id.startsWith("09");
      }))
      .enter().append("circle")
      .attr("class", "bubbles")
      .attr("r", function(d) { return 0; })
      .attr("cx", function(d) { return path.centroid(d)[0] })
      .attr("cy", function(d) { return path.centroid(d)[1] })
      .style("fill", "#900")
      .on("mouseover", handleMouseOver)
      .on("mouseout", handleMouseOut);
}

function redraw() {

  d3.selectAll(".counties")
      .selectAll("path").transition().duration(50)
      .attr("fill", function(d) {
          initCovidd = covid_deaths.get(at_rate).get(at_date).get(d.id);
          if (initCovidd === undefined || initCovidd < 1) {
            value = 0;
          } else {
            value = initCovidd;
          }
          return color(value);
      });

  d3.selectAll(".bubbles")
      .transition().duration(2000)
      .attr("r", function(d) { return rad(d.id); });
}
</script>
