<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";

  let data = [];

  onMount(async () => {
    const res = await fetch('IU_US.csv');
    const text = await res.text();
    data = d3.csvParse(text);

    // Parse the data for the scatter plot
    data.forEach(d => {
      d["Avg. Event Capacity"] = +d["Avg. Event Capacity"];
      d["Avg. Gross USD"] = +d["Avg. Gross USD"];
    });

    createScatterPlot(data);
  });

  function createScatterPlot(data) {
    const width = 800;
    const height = 500;
    const margin = { top: 20, right: 30, bottom: 40, left: 80 }; // Increased left margin

    // Create SVG element
    const svg = d3.select("#scatterplot")
      .append("svg")
      .attr("width", width)
      .attr("height", height);

    // Create scales
    const x = d3.scaleLinear()
      .domain([0, d3.max(data, d => d["Avg. Event Capacity"])])
      .range([margin.left, width - margin.right]);

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => d["Avg. Gross USD"])])
      .range([height - margin.bottom, margin.top]);

    // Create axes
    const xAxis = d3.axisBottom(x);
    const yAxis = d3.axisLeft(y)
      .tickFormat(d => `$${d3.format(".2s")(d).replace("G", "B")}`);


    svg.append("g")
      .attr("transform", `translate(0,${height - margin.bottom})`)
      .call(xAxis)
      .attr("font-size", '14px');

    svg.append("g")
      .attr("transform", `translate(${margin.left},0)`) // Adjust for the new margin
      .call(yAxis)
      .attr("font-size", '14px');

    // Create tooltip element
    const tooltip = d3.select("body").append("div")
      .attr("class", "tooltip")
      .style("position", "absolute")
      .style("background-color", "white")
      .style("border", "1px solid black")
      .style("padding", "5px")
      .style("display", "none");

    // Add dots with tooltip interaction
    svg.selectAll("circle")
      .data(data)
      .enter()
      .append("circle")
      .attr("cx", d => x(d["Avg. Event Capacity"]))
      .attr("cy", d => y(d["Avg. Gross USD"]))
      .attr("r", 5)
      .attr("fill", "blue")
      .on("mouseover", function (event, d) {
        tooltip.style("display", "block")
          .html(`Venue: ${d.Venue}<br>Avg. Gross USD: $${d["Avg. Gross USD"].toLocaleString()}`);
      })
      .on("mousemove", function (event) {
        tooltip.style("top", (event.pageY + 10) + "px")
          .style("left", (event.pageX + 10) + "px");
      })
      .on("mouseout", function () {
        tooltip.style("display", "none");
      });
  }
</script>

<main>
  <h1>Scatter Plot of Avg. Gross USD vs. Avg. Event Capacity</h1>
  <div id="scatterplot"></div>
</main>

<style>
  #scatterplot {
    max-width: 800px;
    margin: 0 auto;
  }

  .tooltip {
    position: absolute;
    background-color: white;
    border: 1px solid black;
    padding: 5px;
    display: none;
  }
</style>
