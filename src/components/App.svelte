<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svg;
  let data = new Map();
  let colorScale;

  let internet_usage = []
  let countryISO_map = new Map();
  let selectedYear = 2010; // Default year

  onMount(async () => {
    // Fetch data
    const [geoJSON, csvData] = await Promise.all([
      d3.json("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world.geojson"),
      d3.csv("https://raw.githubusercontent.com/holtzy/D3-graph-gallery/master/DATA/world_population.csv")
    ]);

    const res = await fetch("internet_usage.csv");
    const csv = await res.text();
    internet_usage = d3.csvParse(csv, d3.autoType)

    // Map country names to ISO codes
    geoJSON.features.forEach((element) => countryISO_map.set(element.properties.name, element.id));

    // Process CSV data and store it in the Map
    internet_usage.forEach(d => {
      if (countryISO_map.has(d.name)) { 
        if(!data.has(d.year)){
          data.set(d.year, new Map());
        }
        data.get(d.year).set(countryISO_map.get(d.name), d.value);
      }
    });

    console.log(countryISO_map);
    console.log(data);
    // Setup SVG
    svg = d3.select("svg");
    var width = +svg.attr("width"), 
        height = +svg.attr("height");
    // Map and projection
    const path = d3.geoPath();
    const projection = d3.geoEquirectangular()
      .scale(150)
      .center([0,0])
      .translate([width/2 , height/2]);

    // Color scale
    colorScale = d3.scaleThreshold()
      .domain([.1,10,20,60,80,1])
      .range(d3.schemeBlues[6]);

    // Draw the map
    svg.append("g")
      .selectAll("path")
      .data(geoJSON.features)
      .enter()
      .append("path")
      // draw each country
      .attr("d", d3.geoPath().projection(projection))
      // set the color of each country
      .attr("fill", d => {
        const total = data.get(selectedYear).get(d.id) || 0;
        return colorScale(total);
      })
      .style("stroke", "transparent")
      .attr("class", "Country")
      .style("opacity", 0.8)
      .on("mouseover", function() {
        d3.select(this)
          .transition()
          .duration(200)
          .style("opacity", 1)
          .style("stroke", "black");
      })
      .on("mouseleave", function() {
        d3.select(this)
          .transition()
          .duration(200)
          .style("opacity", 0.8)
          .style("stroke", "transparent");
      });
  });

  // Reactive statement moved to top-level
  $: {
    if (data.has(selectedYear)) {
      updateMap();
    }
  }

  // Function to update the map
  function updateMap() {
    svg.selectAll("path")
      .attr("fill", d => {
        const total = data.get(selectedYear).get(d.id) || 0;
        return colorScale(total);
      });
  }
</script>

<main>
  <svg width="1000" height="500"></svg>
  <div class="overlay">
    <label>{selectedYear}</label>
    <input
      id="slider"
      type="range"
      min="2000"
      max="2021"
      bind:value ={selectedYear}
    />
  </div>
</main>
