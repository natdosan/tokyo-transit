<script>
  import { onMount } from 'svelte';
  import mapboxgl from 'mapbox-gl';

  mapboxgl.accessToken = 'pk.eyJ1IjoibmF0ZG9zYW4iLCJhIjoiY2xza3huODg4MDh1ZzJpcDVoOTJ1eWFqayJ9.hvQnPf9rwTCV4aok1j7xJA';

  let mapTokyo;
  let isRailwayVisible = false;
  let areStationsVisible = false; // Initial state
  let isChoroplethVisible = false; 
  let stationMarkers = [];
  let isTopTextVisible = true;
  let isBottomTextVisible = false;

  onMount(() => {
    // Tokyo map
    mapTokyo = new mapboxgl.Map({
      container: 'map-tokyo',
      style: 'mapbox://styles/mapbox/navigation-night-v1',
      center: [139.7528, 35.6852], // 35.6852° N, 139.7528° E
      zoom: 12
    });

    const popup = new mapboxgl.Popup({
      closeButton: false,
      closeOnClick: false
    });

    // Add railways
    mapTokyo.on('load', async() => {
      mapTokyo.addSource('tokyo-railways', {
        type: 'geojson',
        data: '/N02-19_RailroadSection.geojson'
      });

      // Load Population Density Choropleth source
      mapTokyo.addSource('tokyo-population-density', {
          type: 'geojson',
          data: '/tokyo23_population.geojson'
      });

      // Load administrative boundaries
      mapTokyo.addSource('tokyo-boundaries', {
        type: 'geojson',
        data: '/tokyo23_population.geojson'
      });

      mapTokyo.addLayer({
        id: 'tokyo-railways-layer',
        type: 'line',
        source: 'tokyo-railways',
        layout: {},
        paint: {
          'line-color': '#ADD8E6',
          'line-width': 2,
          'line-opacity': isRailwayVisible ? 1 : 0
        }
      });

      mapTokyo.on('mousemove', 'tokyo-railways-layer', (e) => {
        if (e.features.length > 0) {
          const feature = e.features[0];
          popup.setLngLat(e.lngLat)
               .setText(feature.properties.路線名)
               .addTo(mapTokyo);
        }
      });

      mapTokyo.on('mouseleave', 'tokyo-railways-layer', () => {
        popup.remove();
      });

      // add administrative boundary layer
      mapTokyo.addLayer({
        id: 'tokyo-administrative-boundaries',
        type: 'line',
        source: 'tokyo-boundaries',
        layout: {},
        paint: {
          'line-color': '#FFFFFF',
          'line-width': 2,
        }
      });

      // add the population density layer
      mapTokyo.addLayer({
        id: 'tokyo-population-density',
        type: 'fill',
        source: 'tokyo-population-density',
        paint: {
          'fill-color': [
            'interpolate',
            ['linear'],
            ['get', 'population'],
            0, '#ffffff',
            100000, '#614144',
            300000, '#802A37',
            500000, '#A51C2A',
            700000, '#DE0D0D',
            1000000, '#FF9641'
          ],
          'fill-opacity': 0.5
        }
      });

      mapTokyo.on('mousemove', 'tokyo-population-density', (e) => {
        if (e.features.length > 0) {
          const feature = e.features[0];
          const population = feature.properties.population; // Assuming there's a 'population' property in your geojson data
          const wardName = feature.properties.N03_004;
          const popupText = population ? `${wardName} Population: ${population}` : 'No population data available';
          popup.setLngLat(e.lngLat)
              .setText(popupText)
              .addTo(mapTokyo);
        }
      });

      const response = await fetch('/stations.json'); 
      const stationsData = await response.json(); 

      Object.values(stationsData).flat().forEach(station => {
        const score = 1 - (station.ridership / station.population);
        const marker = new mapboxgl.Marker()
          .setLngLat(station.coordinates)
          .setPopup(new mapboxgl.Popup().setText(`${station.name}: Daily Ridership - ${station.ridership}, \n Score: ${score.toFixed(2)}`))
          .addTo(mapTokyo);

        stationMarkers.push(marker); // Store the marker reference
      });
    });
  });

  // Reactive Elements
  function toggleRailwayVisibility() {
    isRailwayVisible = !isRailwayVisible;
    mapTokyo.setPaintProperty('tokyo-railways-layer', 'line-opacity', isRailwayVisible ? 1 : 0);
  }
  function toggleStationVisibility() {
    areStationsVisible = !areStationsVisible;
    stationMarkers.forEach(marker => {
      areStationsVisible ? marker.addTo(mapTokyo) : marker.remove();
  });
  }
  // Toggle choropleth visibility
  function toggleChoroplethVisibility() {
    isChoroplethVisible = !isChoroplethVisible;
    mapTokyo.setLayoutProperty('tokyo-population-density', 'visibility', isChoroplethVisible ? 'visible' : 'none');
  }

  // Function to toggle the visibility of the bottom text
  function toggleBottomTextVisibility() {
    isBottomTextVisible = !isBottomTextVisible;
    const bottomTextDiv = document.getElementById('bottom-text');
    bottomTextDiv.style.display = isBottomTextVisible ? 'block' : 'none';
  }
</script>

<style>
  /* Styles for top and bottom text */
  .top-text, .bottom-text {
    padding: 20px; /* Increase padding for a more spacious feel */
    text-align: center; /* Center align text */
    background-color: #222; /* Dark background color for a modern look */
    color: #fff; /* White text color for better contrast */
    font-family: 'Roboto', sans-serif; /* Modern font */
  }

  /* Add a subtle shadow for depth */
  .top-text, .bottom-text {
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1); 
  }

  /* Add rounded corners for a softer appearance */
  .top-text, .bottom-text {
    border-radius: 0px; 
  }

  .top-text {
    padding: 20px;
    text-align: center;
    background-color: #222;
    color: #fff;
    font-family: 'Roboto', sans-serif;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    border-radius: 0px;
  }

  .bottom-text {
    padding: 20px;
    text-align: center;
    background-color: #222;
    color: #fff;
    font-family: 'Roboto', sans-serif;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    border-radius: 0px;
  }
  /* Style for main content container */
  .main-content {
    display: flex;
    flex-direction: column;
    align-items: center; /* Center align content horizontally */
    justify-content: space-between; /* Align content with space between */
    min-height: 100vh; /* Ensure the container takes at least the full viewport height */
    position: relative; /* Positioning context for absolute positioning */
  }

  /* Style for the map container */
  #map-tokyo {
    flex: 1; /* Allow map container to grow and occupy remaining space */
    width: 100%; /* Ensure map container takes full width */
    position: relative; /* Positioning context for absolute positioning */
    border-radius: 0px; 
  }

  /* Style for toggle buttons */
  .toggle-button-container {
    position: absolute;
    top: 20px; /* Adjust top position as needed */
    right: 20px; /* Adjust right position as needed */
    z-index: 10; /* Ensure buttons stay above map layers */
  }

  /* Style for individual toggle buttons */
  .toggle-button {
    margin-bottom: 10px; /* Add some spacing between buttons */
  }

  .legend {
    position: absolute;
    bottom: 20px;
    left: 20px;
    background-color: rgba(255, 255, 255, 0.8);
    border-radius: 5px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.2);
    padding: 5px;
    z-index: 11; /* Ensure buttons stay above map layers */
  }

  .legend-item {
    display: flex;
    justify-content: space-between;
    margin-bottom: 5px;
    padding: 2px 5px;
    font-size: 12px;
    z-index: 11; /* Ensure buttons stay above map layers */
  }
</style>


<div class="main-content">
  <!-- Top text -->
  <div id="top-text" class="top-text" style="{isTopTextVisible ? 'display: block;' : 'display: none;'}">
    <h1>Welcome to Tokyo Map</h1>
    <p>Have you ever gotten lost in a massive city and couldn't find the closest train station to get to your next destination? 
      Well, we got the solution for you! Introducing the Tokyo Map! Our motivation for this visualization is to provide the user 
      an easy-to-use map with customization, giving the user a map with plenty of digestible information while keeping it clear and understandable.
      We provided togglable metro stations, metro lines, as well as population ridership through geoJSON data with the click of a button; 
      specifically, popularly used maps exclude ridership and population data, but in our map, it has a togglable feature layer in the form of a choropleth. 
    </p>
  </div>

  <!-- Map container -->
  <div id="map-tokyo">
    <!-- Button container -->
    <div class="toggle-button-container">
      <button class="toggle-button" on:click={toggleRailwayVisibility}>
        {isRailwayVisible ? 'Hide Railway Lines' : 'Show Railway Lines'}
      </button>
      <button class="toggle-button" on:click={toggleStationVisibility}>
        {areStationsVisible ? 'Hide Stations' : 'Show Stations'}
      </button>
      <button class="toggle-button" on:click={toggleChoroplethVisibility}>
        {isChoroplethVisible ? 'Hide Choropleth' : 'Show Choropleth'}
      </button>
      <button class="toggle-button" on:click={toggleBottomTextVisibility}>
        {isBottomTextVisible ? 'Hide Bottom Text' : 'Learn More!'}
      </button>
    </div>

    <div class="legend">
      <div class="legend-item" style="background-color: #ffffff;">0</div>
      <div class="legend-item" style="background-color: #614144;">100,000</div>
      <div class="legend-item" style="background-color: #802A37;">300,000</div>
      <div class="legend-item" style="background-color: #A51C2A;">500,000</div>
      <div class="legend-item" style="background-color: #DE0D0D;">700,000</div>
      <div class="legend-item" style="background-color: #FF9641;">1,000,000</div>
    </div>
  </div>



  <!-- Bottom text -->
  <div id="bottom-text" class="bottom-text" style="{isBottomTextVisible ? 'display: block;' : 'display: none;'}">
    <p>Using a choropleth allows an easy-to-interpret representation of Tokyo's population density by ward, making it unique, promoting exploration throughout the city 
      without worry. Additionally, it allows the user to toggle between showing and hiding the 10 most popular railway stations and lines, allowing for customizability 
      and a clean user-experience. An interesting observation of the visualization is that we can understand the population density and ridership of each railway 
      station and area which helps evaluate the busyness of each station. Specifically, we designed a metric where we subtract the ratio of riders to the ward's 
      relative population density from one, which gives us a negative/positive scalar, where negative scores represent a more crowded area while positive scores
      represent a less crowded area. We chose this compared to a log of the score because the trend in our data did not seem to be exponential. Together, the population 
      chorepleth, railway, live traffic feed, and carefully designed metro station scores allow for a successful story to be told through the visualization. Each station is located
      in its own ward, and has its own daily ridership statistic, but without taking into account ward's population density, it's hard to tell just how packed a station 
      will typically be. Therefore, the positive and negative scores for each station allow a rider to predict how busy a station will be. 
      Furthermore, the main takeaway is that we can use this metric to help users predict if their destination will be crowded and factor in commute times, which can help users to 
      avoid busy stations if possible as well as help individuals with intense social anxiety avoid super crowded stations. And that is the Tokyo map!
    </p>
    <button class="toggle-button" on:click={toggleBottomTextVisibility}>
      {isBottomTextVisible ? 'Hide Bottom Text' : 'Show Bottom Text'}
    </button>
  </div>
</div>
