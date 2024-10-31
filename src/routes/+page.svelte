<style>
    @import url("$lib/global.css");
</style>

<script>
    import { onMount } from "svelte";
    import * as d3 from 'd3';
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";

    mapboxgl.accessToken = "pk.eyJ1IjoiamFkZWRlbyIsImEiOiJjbTJuZzFpYWkwNTdhMmlvbW16bmt3bjlhIn0.QCsOvTO9JomVioOyAgZgPA";

    let map;
    let mapViewChanged = 0;
    let stations = [];
    let trips = [];

    let timeFilter = -1;

    $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter).toLocaleString("en", {timeStyle: "short"});

    let radiusScale;
    $: radiusScale = d3.scaleSqrt()
        .domain([0, d3.max(stations, d => d.totalTraffic)])
        .range(timeFilter == -1 ? [0, 25] : [3, 50]);

    let stationFlow = d3.scaleQuantize()
        .domain([0, 1])
        .range([0, 0.5, 1]);

    async function loadMap() {
        map = new mapboxgl.Map({
            container: 'map',
            center: [-79.94557, 40.44362],
            zoom: 12,
            style: "mapbox://styles/mapbox/streets-v12"
	    });

        //wait until map is loaded before adding more data
        await new Promise(resolve => map.on("load", resolve));

        map.addSource("pgh_route", {
            type: "geojson",
            data: "https://data.wprdc.org/dataset/407de508-f9dc-4e39-94ed-aeadceddcaea/resource/cf7fc05d-be6f-46ad-82e4-9559fc0a306a/download/pittsburghbikeinfrastructure.geojson",
        });

        map.addLayer({
            id: "pgh_route_layer",
            type: "line",
            source: "pgh_route",
            paint: {
                "line-color": "green",
                "line-width": 2,
                "line-opacity": 0.4,
            },
        });

        stations =  await d3.csv("https://dig.cmu.edu/datavis-fall-2024/labs/8/data/pgh_bike_stations.csv");

        trips = await d3.csv("https://dig.cmu.edu/datavis-fall-2024/labs/8/data/pgh_bike_traffic_2024-09.csv").then(trips => {
            for (let trip of trips) {
                trip.started_at = new Date(trip["Start Date"]);
                trip.ended_at = new Date(trip["End Date"]);
            }
            return trips;
        });
    }

    function minutesSinceMidnight (date) {
        return date.getHours() * 60 + date.getMinutes();
    }

    function getCoords (station) {
        let point = new mapboxgl.LngLat(+station.Longitude, +station.Latitude);
        let {x, y} = map.project(point);
        return {cx: x, cy: y};
    }

    let arrivals;
    let departures;
    
    onMount(() => {
	    loadMap().then(() => {
            departures = d3.rollup(trips, v => v.length, d => d["Start Station Id"]);
            arrivals = d3.rollup(trips, v => v.length, d => d["End Station Id"]);

            stations = stations.map(station => {
                let id = station.Id;
                station.arrivals = arrivals.get(id) ?? 0;
                station.departures = departures.get(id) ?? 0;
                
                station.totalTraffic = station.arrivals + station.departures;

                return station;
            });
        });    
    })

    let filteredTrips = [];
    let filteredArrivals = [];
    let filteredDepartures = [];
    let filteredStations = [];

    $: filteredTrips = timeFilter === -1? trips : trips.filter(trip => {
        let startedMinutes = minutesSinceMidnight(trip.started_at);
        let endedMinutes = minutesSinceMidnight(trip.ended_at);
        return Math.abs(startedMinutes - timeFilter) <= 60
            || Math.abs(endedMinutes - timeFilter) <= 60;
    });

    $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d["Start Station Id"]);
    $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d["End Station Id"]);

    $: filteredStations = stations.map(station => {
        station = {...station};
        let id = station.Id;
        station.arrivals = filteredArrivals.get(id) ?? 0;
        station.departures = filteredDepartures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;

        return station;
    })
        
    //event listener that adds 1 to mapViewChanged when map is moved
    $: map?.on("move", evt => mapViewChanged++);

</script>

<div id="header">
    <img alt="person biking emoji" id="bike-favicon" src="/favicon.svg" height="30"/>
    <h1>Bikewatching</h1>
</div>

<div>
    <label> Filter by time:
        <input type=range min="-1" max="1440" bind:value={timeFilter}>
        <time datetime={timeFilter}>
            {#if timeFilter == -1}
                <em>(any time)</em>
            {:else}
                 {timeFilterLabel}
            {/if}
        </time>
        </label>
</div>

<div class="legend">
	<div class="colorCode" style="--departure-ratio: 1">More departures</div>
	<div class="colorCode" style="--departure-ratio: 0.5">Balanced</div>
	<div class="colorCode" style="--departure-ratio: 0">More arrivals</div>
</div>

<div id="map">
    <svg>
        {#key mapViewChanged}
            {#each filteredStations as station}
                <circle
                    cx={getCoords(station).cx}
                    cy={getCoords(station).cy}
                    r={radiusScale(station.totalTraffic)}
                    fill="steelblue"
                    style="--departure-ratio: { stationFlow(station.departures / station.totalTraffic) }"
                    class="colorCode"
                >
                    
                    <title>{station.totalTraffic} trips ({station.departures} departures, { station.arrivals} arrivals)</title>
                </circle>
            {/each}
        {/key}
    </svg>
</div>







