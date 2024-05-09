<script>
    import mapboxgl from "mapbox-gl";
    import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
    import { onMount } from "svelte";
    import * as d3 from "d3";
    export const prerender = true;

    let map;
    let stations = [];
    let trips = [];
    let departures = [];
    let arrivals = [];
    let timeFilter = -1;
    let filteredTrips = [];
    let mapViewChanged = 0;
    let departuresByMinute = Array.from({ length: 1440 }, () => []);
    let arrivalsByMinute = Array.from({ length: 1440 }, () => []);
    $: map?.on("move", (evt) => mapViewChanged++);
    let filteredDepartures = [];
    let filteredArrivals = [];
    let stationFlow = d3.scaleQuantize().domain([0, 1]).range([0, 0.5, 1]);
    //
    mapboxgl.accessToken =
        "pk.eyJ1IjoiYnVya2VkbSIsImEiOiJjbHViOXBobzQwdG8wMmxwNnk2dDJhMmp6In0.jyh_zboCAZPmR-dn5GcdVA";

    onMount(async () => {
        map = new mapboxgl.Map({
            container: "map",
            style: "mapbox://styles/mapbox/streets-v11",
            center: [-71.09451, 42.36027],
            zoom: 13,
        });

        await new Promise((resolve) => map.on("load", resolve));

        map.addSource("boston_route", {
            type: "geojson",
            data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
        });

        map.addSource("cambridge_route", {
            type: "geojson",
            data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson",
        });

        map.addLayer({
            id: "boston_lanes", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "boston_route", // The id we specified in `addSource()`
            paint: {
                "line-color": "green", // Set the line color to a nice green
                "line-width": 3, // Set the line thickness to 10
                "line-opacity": 0.4,
            },
        });

        map.addLayer({
            id: "cambridge_lanes", // A name for our layer (up to you)
            type: "line", // one of the supported layer types, e.g. line, circle, etc.
            source: "cambridge_route", // The id we specified in `addSource()`
            paint: {
                "line-color": "green", // Set the line color to a nice green
                "line-width": 3, // Set the line thickness to 10
                "line-opacity": 0.4,
            },
        });

        stations = await d3.csv(
            "https://vis-society.github.io/labs/8/data/bluebikes-stations.csv",
        );

        let TRIP_DATA_URL =
            "https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv";

        trips = await d3.csv(TRIP_DATA_URL).then((trips) => {
            for (let trip of trips) {
                trip.started_at = new Date(trip.started_at);
                trip.ended_at = new Date(trip.ended_at);
                let startMinute = minutesSinceMidnight(trip.started_at);
                let endMinute = minutesSinceMidnight(trip.ended_at);
                departuresByMinute[startMinute].push(trip);
                arrivalsByMinute[endMinute].push(trip);
            }
            return trips;
        });

        arrivals = d3.rollup(
            trips,
            (v) => v.length,
            (d) => d.end_station_id,
        );

        departures = d3.rollup(
            trips,
            (v) => v.length,
            (d) => d.start_station_id,
        );

        stations = stations.map((station) => {
            let id = station.Number;
            station.arrivals = arrivals.get(id) ?? 0;
            station.departures = departures.get(id) ?? 0;
            station.totalTraffic = station.arrivals + station.departures;
            return station;
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
    $: radiusScale = d3
        .scaleSqrt()
        .domain([0, d3.max(stations, (d) => d.totalTraffic)])
        .range([0, 25]);
    departures = d3.rollup(
        trips,
        (v) => v.length,
        (d) => d.start_station_id,
    );
    $: timeFilterLabel =
        timeFilter === -1
            ? "(any time)"
            : new Date(0, 0, 1, 0, timeFilter).toLocaleString("en", {
                  timeStyle: "short",
              });

    $: filteredTrips =
        timeFilter === -1
            ? trips
            : trips.filter((trip) => {
                  let startedMinutes = minutesSinceMidnight(trip.started_at);
                  let endedMinutes = minutesSinceMidnight(trip.ended_at);
                  return (
                      Math.abs(startedMinutes - timeFilter) <= 60 ||
                      Math.abs(endedMinutes - timeFilter) <= 60
                  );
              });

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
    $: filteredDepartures = filterByMinute(departuresByMinute, timeFilter);
    $: filteredArrivals = filterByMinute(arrivalsByMinute, timeFilter);
    $: filteredDeparturesAggregate = d3.rollup(
        filteredDepartures,
        (v) => v.length,
        (d) => d.start_station_id,
    );
    $: filteredArrivalsAggregate = d3.rollup(
        filteredArrivals,
        (v) => v.length,
        (d) => d.end_station_id,
    );
    $: stations =
        timeFilter === -1
            ? stations
            : stations.map((station) => {
                  station = { ...station }; // Clone to avoid mutating the original
                  let id = station.Number;
                  station.departures = filteredDeparturesAggregate.get(id) ?? 0;
                  station.arrivals = filteredArrivalsAggregate.get(id) ?? 0;
                  station.totalTraffic = station.departures + station.arrivals;
                  return station;
              });
</script>

<h1>Boston Bike Routes</h1>

<div class="row">
    Filter by time:
    <input type="range" min="-1" max="1440" bind:value={timeFilter} />
    <time>
        {timeFilterLabel}
    </time>
</div>
<div id="map">
    {#key mapViewChanged}
        <svg>
            {#each stations as station}
                <circle
                    {...getCoords(station)}
                    r={radiusScale(station.totalTraffic)}
                    style="--departure-ratio: {stationFlow(
                        station.departures / station.totalTraffic,
                    )}"
                >
                    <title>
                        {station.totalTraffic} trips ({station.departures} departures,
                        {station.arrivals} arrivals)
                    </title>
                </circle>
            {/each}
        </svg>
    {/key}
</div>

<div class="legend">
    <div>Legend:</div>
    <div
        class="color-swatch"
        style="--departure-ratio: 1; background: color-mix(in oklch, var(--color-departures) calc(100% * 1), var(--color-arrivals));"
    ></div>
    <div>More departures</div>
    <div
        class="color-swatch"
        style="--departure-ratio: 0.5; background: color-mix(in oklch, var(--color-departures) 50%, var(--color-arrivals) 50%);"
    ></div>
    <div>Balanced</div>
    <div
        class="color-swatch"
        style="--departure-ratio: 0; background: var(--color-arrivals);"
    ></div>
    <div>More arrivals</div>
</div>

<style>
    @import url("$lib/global.css");

    :root {
        --color-departures: steelblue;
        --color-arrivals: darkorange;
    }

    .legend {
        display: flex;
        justify-content: center;
        align-items: center;
        gap: 10px; /* Spacing between elements */
    }
    .row {
        display: flex;
        flex-direction: row;
    }

    .color-swatch {
        width: 20px;
        height: 20px;
        border: 1px solid #ccc;
        background: var(--color);
    }

    #map svg {
        position: absolute;
        z-index: 1;
        width: 100%;
        height: 100%;
        pointer-events: none;
    }

    circle {
        fill-opacity: 0.6;
        stroke: white;
        --color-departures: steelblue;
        --color-arrivals: darkorange;
        --color: color-mix(
            in oklch,
            var(--color-departures) calc(100% * var(--departure-ratio)),
            var(--color-arrivals)
        );
        fill: var(--color);
    }
</style>
