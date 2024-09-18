<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";

  let data = [];
  let data2 = [];
  let combinedData = [];
  let selectedArtist = "IU"; // Bind this to the dropdown for artist selection
  let sortedArtists = []; // To store alphabetically sorted artist names
  let circles; // Define a variable to store the circle elements

  onMount(async () => {
    const res = await fetch('Pollstar_concert_k-pop.csv');
    const res2 = await fetch('IU_US.csv');
    const text = await res.text();
    const text2 = await res2.text();
    data = d3.csvParse(text);
    data2 = d3.csvParse(text2);

    combinedData = data.concat(data2);

    // Filter for 'United States' and drop the 'Country' column
    combinedData = combinedData.filter(d => d['Country'] === 'United States');
    combinedData.forEach(d => delete d['Country']);

    // Replace 'Ticket Price Max' where it's 0 with 'Ticket Price Min'
    combinedData.forEach(d => {
      if (+d['Ticket Price Max'] === 0) {
        d['Ticket Price Max'] = d['Ticket Price Min'];
      }
    });

    // Clean and convert 'Avg. Event Capacity', 'Ticket Price Min', 'Ticket Price Max', and 'Avg. Gross USD' to numerical values
    combinedData.forEach(d => {
      d['Avg. Event Capacity'] = +d['Avg. Event Capacity'].replace(/,/g, '');
      d['Ticket Price Min'] = +d['Ticket Price Min'].replace(/,/g, '');
      d['Ticket Price Max'] = +d['Ticket Price Max'].replace(/,/g, '');
      d['Avg. Gross USD'] = +d['Avg. Gross USD'].replace(/\$/g, '').replace(/,/g, '');
    });

    // Parse 'Event Date' and extract 'day_of_week' and 'month' as categorical values
    combinedData.forEach(d => {
      const eventDate = new Date(d['Event Date']);
      d['day_of_week'] = eventDate.getDay();
      d['month'] = eventDate.getMonth() + 1;
    });

    // Sort the artist names alphabetically 
    sortedArtists = Array.from(new Set(combinedData.map(d => d.Headliner)))
      .sort((a, b) => a.localeCompare(b));

    createScatterPlot(combinedData);
  });

  function createScatterPlot(data) {
    const width = 800;
    const height = 500;
    const margin = { top: 20, right: 30, bottom: 40, left: 80 };

    const svg = d3.select("#scatterplot")
      .append("svg")
      .attr("width", width)
      .attr("height", height);

    const x = d3.scaleLinear()
      .domain([0, d3.max(data, d => d["Avg. Event Capacity"])])
      .range([margin.left, width - margin.right]);

    const y = d3.scaleLinear()
      .domain([0, d3.max(data, d => d["Avg. Gross USD"])])
      .range([height - margin.bottom, margin.top]);

    const xAxis = d3.axisBottom(x);
    const yAxis = d3.axisLeft(y)
      .tickFormat(d => `$${d3.format(".2s")(d).replace("G", "B")}`);

    svg.append("g")
      .attr("transform", `translate(0,${height - margin.bottom})`)
      .call(xAxis)
      .attr("font-size", '14px');

    svg.append("g")
      .attr("transform", `translate(${margin.left},0)`)
      .call(yAxis)
      .attr("font-size", '14px');

    const tooltip = d3.select("body").append("div")
      .attr("class", "tooltip")
      .style("position", "absolute")
      .style("background-color", "white")
      .style("border", "1px solid black")
      .style("padding", "5px")
      .style("display", "none");

    // Append circles and store the selection in the `circles` variable
    circles = svg.selectAll("circle")
      .data(data)
      .enter()
      .append("circle")
      .attr("cx", d => x(d["Avg. Event Capacity"]))
      .attr("cy", d => y(d["Avg. Gross USD"]))
      .attr("r", 2.5)
      .attr("fill", "blue")
      .on("mouseover", function (event, d) {
        tooltip.style("display", "block")
          .html(`Venue: ${d.Venue}<br>Avg. Gross USD: $${d["Avg. Gross USD"].toLocaleString()}<br>Headliner: ${d.Headliner}`);
      })
      .on("mousemove", function (event) {
        tooltip.style("top", (event.pageY + 10) + "px")
          .style("left", (event.pageX + 10) + "px");
      })
      .on("mouseout", function () {
        tooltip.style("display", "none");
      });
  }

  // Reactive block to highlight points for the selected artist
  $: if (circles) {
    circles.attr("fill", d => d.Headliner === selectedArtist ? "red" : "gray")
    .filter(d => d.Headliner === selectedArtist)
    .raise();
  }
</script>

<main>
  <h1>Scatter Plot of Avg. Gross USD vs. Avg. Event Capacity</h1>
  
  <!-- Searchable and alphabetically ordered Artist Dropdown -->
  <input list="artist-list" bind:value={selectedArtist} placeholder="Search artist..." />
  <datalist id="artist-list">
    {#each sortedArtists as artist}
      <option value={artist}>{artist}</option>
    {/each}
  </datalist>

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
