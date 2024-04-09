<style>
    @import url("$lib/global.css");
</style>
<header>
    <h1>
        Bikewatching
    </h1>
    <label for="time-filter">Filter by time:</label>
    <input type="range" id="time-filter" name="time-filter" min="-1" max="1440" bind:value={timeFilter}>
    <time id="time-display">{timeFilterLabel}</time>
    {#if timeFilter === -1}
        <em>(any time)</em>
    {/if}
</header>


<div id="map">
    <svg>
        {#each stations as station}

            {#key mapViewChanged}
                <circle { ...getCoords(station) } r={radiusScale(station.totalTraffic)} fill="steelblue" />
            {/key}
            
        {/each}
    </svg>
</div>

<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import * as d3 from "d3";    
    mapboxgl.accessToken = "pk.eyJ1IjoicmlsZXlvaDYiLCJhIjoiY2x1cnN5MnJmMDl5aTJrb2FxcnVjNHplMiJ9.UET4UlquctNJFvnZKFrDWQ"
    import { onMount } from "svelte";
    import { filter } from "d3";

    let timeFilter = -1;
    $: timeFilterLabel = timeFilter !== -1 ? new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {timeStyle: "short"}) : '';

    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    function minutesSinceMidnight (date) {
	    return date.getHours() * 60 + date.getMinutes();
    }

    let mapViewChanged = 0;
    $: map?.on("move", evt => mapViewChanged++);

    let stations = [];
    let trips = [];
    let departures;
    let arrivals;
    let map;
    onMount(async () => {
    map = new mapboxgl.Map({
        style: "mapbox://styles/mapbox/streets-v12",
        container: "map",
        center: [-71.09415, 42.36027],
        zoom: 12
    });

    await new Promise(resolve => map.on("load", resolve));

    map.addSource("boston_route", {
        type: "geojson",
        data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
    });
    map.addSource("cambridge_route", {
        type: "geojson",
        data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
    });

    map.addLayer({
        id: 'boston routes',
        type: "line",
        source: "boston_route",
        paint: {
            "line-color": "green",
            "line-width": 2,
            "line-opacity": 0.5
	    },
    });
    map.addLayer({
        id: 'cambridge routes',
        type: "line",
        source: "cambridge_route",
        paint: {
            "line-color": "green",
            "line-width": 2,
            "line-opacity": 0.5
	    },
    });
    stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");

    trips = await d3.csv('https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv').then(trips => {
    for (let trip of trips) {
        trip.started_at = new Date(trip.started_at);
        trip.ended_at = new Date(trip.ended_at);
    }
    return trips;
    });
    departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
    arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);
    stations = stations.map(station => {
        station = {...station};
        let id = station.Number;
        station.arrivals = arrivals.get(id) ?? 0;
        station.departures = departures;
        station.totalTraffic = station.arrivals + station.departures.size;
        return station;
    });
});

$: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic)])
	.range([0, 25]);

$: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
	let startedMinutes = minutesSinceMidnight(trip.started_at);
	let endedMinutes = minutesSinceMidnight(trip.ended_at);
	return Math.abs(startedMinutes - timeFilter) <= 60
	       || Math.abs(endedMinutes - timeFilter) <= 60;
});

</script>
