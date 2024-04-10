<style>
    @import url("$lib/global.css");
    #map {
      height: 400px; /* You can set the height as needed */
    }
  
    #map svg {
      position: absolute;
      z-index: 1;
      width: 100%;
      height: 100%;
      pointer-events: none;
    }
  
    #map svg circle {
      --color-departures: steelblue;
      --color-arrivals: darkorange;
      --color: color-mix(
        in oklch,
        var(--color-departures) calc(100% * var(--departure-ratio)),
        var(--color-arrivals)
      );
      fill: var(--color);
      fill-opacity: 0.6; /* Adjust as needed */
      stroke: white;
      stroke-width: 1;
    }
  
    header {
      display: flex;
      gap: 1em;
      align-items: baseline;
    }
  
    header label {
      margin-left: auto;
    }
  
    header time,
    header em {
      display: block;
    }
  
    header em {
      color: #aaa;
      font-style: italic;
    }
  
    .legend {
      display: flex;
      gap: 1px;
      margin-block: 1em;
    }
    
    .legend > div {
        flex: 1;
        padding: 0.5em 1em;
        text-align: center;
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
            );
            color: var(--color);
            /* This is a little more advanced but looks much more like an actual legend. One downside of it is that it’s harder to generalize to more than 3 colors, as it looks like a legend for a categorical variable.
        Uses a ::before pseudo-element with content: "" to create the swatches.
        Uses an additional element for the “Legend:” label
        Each child <div> also uses flexbox
        Make sure the gap on child <div> is significantly smaller than the gap on the parent .legend to create the effect of the swatches being connected to the labels (design principle of proximity).
        */
        
        
    }
  </style>
  
  <script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    mapboxgl.accessToken = "pk.eyJ1IjoieXVuZ3N1bmciLCJhIjoiY2x1c3R0d21pMGxxdDJxdGJmdnVuaGI5NyJ9.o_c3hYxuIteiPKLrXICtfQ";
  
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
  
    const TRIP_DATA_URL = "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv";
    const STATION_DATA_URL = "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv";
  
    let map;
    let stations = [];
    let mapViewChanged = 0;
    let trips = [];
    let radiusScale;
    let timeFilter = -1; // Initialize the filter to -1 (any time)
    let departuresByMinute = Array.from({ length: 1440 }, () => []);
    let arrivalsByMinute = Array.from({ length: 1440 }, () => []);
  
    onMount(async () => {
      map = new mapboxgl.Map({
        container: 'map',
        style: 'mapbox://styles/mapbox/streets-v12',
        center: [-71.0892, 42.3601],
        zoom: 12,
        minZoom: 10,
        maxZoom: 16,
      });
  
      stations = await d3.csv(STATION_DATA_URL);
      console.log(stations); // Log the data to explore its structure
  
      trips = await d3.csv(TRIP_DATA_URL).then(trips => {
        for (let trip of trips) {
          trip.started_at = new Date(trip.started_at);
          trip.ended_at = new Date(trip.ended_at);
          let startedMinutes = minutesSinceMidnight(trip.started_at);
          departuresByMinute[startedMinutes].push(trip);
          let endedMinutes = minutesSinceMidnight(trip.ended_at);
          arrivalsByMinute[endedMinutes].push(trip);
        }
        return trips;
      });
  
      await new Promise(resolve => map.on("load", resolve));
  
      map.addSource("boston_route", {
        type: "geojson",
        data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
      });
  
      map.addLayer({
        id: 'bike-lanes',
        type: 'line',
        source: 'boston_route',
        layout: {},
        paint: {
          "line-color": "blue", // Translucent but dark green color
          "line-width": 3, // Thicker lines
          "line-opacity": 0.4 // 40% opacity for blending with the map
        }
      });
  
      map.addSource("cambridge_route", {
        type: "geojson",
        data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
      });
  
      map.addLayer({
        id: 'bike-lanes-cambridge',
        type: 'line',
        source: 'cambridge_route',
        layout: {},
        paint: {
          "line-color": "blue", // Translucent but dark green color
          "line-width": 3, // Thicker lines
          "line-opacity": 0.4 // 40% opacity for blending with the map
        }
      });
  
      map.on("move", () => {
        mapViewChanged++;
      });
    });
  
    function getCoords(station) {
      let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
      let { x, y } = map.project(point);
      return { cx: x, cy: y };
    }
  
    function minutesSinceMidnight(date) {
      return date.getHours() * 60 + date.getMinutes();
    }
  
    function filterByMinute(tripsByMinute, minute) {
      let minMinute = (minute - 60 + 1440) % 1440;
      let maxMinute = (minute + 60) % 1440;
  
      if (minMinute > maxMinute) {
        let beforeMidnight = tripsByMinute.slice(minMinute);
        let afterMidnight = tripsByMinute.slice(0, maxMinute);
        return beforeMidnight.concat(afterMidnight).flat();
      } else {
        return tripsByMinute.slice(minMinute, maxMinute).flat();
      }
    }
  
    let stationFlow = d3.scaleQuantize()
      .domain([0, 1])
      .range([0, 0.5, 1]);
  
    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
      .toLocaleString("en", { timeStyle: "short" });
  
    $: filteredDepartures = filterByMinute(departuresByMinute, timeFilter);
    $: filteredArrivals = filterByMinute(arrivalsByMinute, timeFilter);
  
    $: filteredStations = stations.map(station => {
      station = { ...station }; // Clone the station object
      station.arrivals = filteredArrivals.filter(trip => trip.end_station_id === station.Number).length;
      station.departures = filteredDepartures.filter(trip => trip.start_station_id === station.Number).length;
      station.totalTraffic = station.arrivals + station.departures;
      station.departureRatio = station.departures / station.totalTraffic;
      return station;
    });
  
    $: radiusScale = d3.scaleSqrt()
      .domain([0, d3.max(filteredStations, d => d.totalTraffic)])
      .range([0, 25]);
  </script>
  
  <header>
    <h1>🚴‍♂️ Map of Bluebikes</h1>
    <label>
      Filter by time:
      <input type="range" min="-1" max="1440" bind:value={timeFilter} />
      <time>{timeFilterLabel}</time>
      {#if timeFilter === -1}
        <em>(any time)</em>
      {/if}
    </label>
  </header>
  
  <div id="map">
    <svg>
      {#key mapViewChanged}
        {#each filteredStations as station}
          <circle
            {...getCoords(station)}
            r={radiusScale(station.totalTraffic)}
            style="--departure-ratio: {stationFlow(station.departureRatio)}"
          />
        {/each}
      {/key}
    </svg>
  </div>
  
  <div class="legend">
    <div style="--departure-ratio: 1">More departures</div>
    <div style="--departure-ratio: 0.5">Balanced</div>
    <div style="--departure-ratio: 0">More arrivals</div>
  </div>