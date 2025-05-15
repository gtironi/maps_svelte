<script>
import mapboxgl from "mapbox-gl";
import "../../node_modules/mapbox-gl/dist/mapbox-gl.css";
import { onMount } from "svelte";
import * as d3 from "d3";

mapboxgl.accessToken = "pk.eyJ1IjoiZ3VzdGF2b3Rpcm9uaSIsImEiOiJjbWFwaDFrdGQwaHp0MmtxNzhhbXRmZXJvIn0.JrUc6BcdlQKAoeRi88lANg";

let map;
let stations = [];
let mapViewChanged = 0;
let trips;

$: map?.on("move", evt => mapViewChanged++);

async function initMap() {
	map = new mapboxgl.Map({
		container: 'map',
		center: [-71.09415, 42.36027],
		zoom: 12,
		style: "mapbox://styles/mapbox/streets-v12",
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
        id: "bikesdoscrias", // A name for our layer (up to you)
        type: "line", // one of the supported layer types, e.g. line, circle, etc.
        source: "boston_route", // The id we specified in `addSource()`
        paint: {
            "line-color": "green",
            "line-width": 3,
            "line-opacity": 0.4,
        },
    });
    map.addLayer({
        id: "bikesdoscrias2", // A name for our layer (up to you)
        type: "line", // one of the supported layer types, e.g. line, circle, etc.
        source: "cambridge_route", // The id we specified in `addSource()`
        paint: {
            "line-color": "green",
            "line-width": 3,
            "line-opacity": 0.4,
        },
    });
}

async function loadStationData() {
    try {
        const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-stations.csv';
        const data = await d3.csv(csvUrl);

        stations = data.map(station => ({
            id: station.Number,
            name: station.NAME,
            Lat: +station.Lat,
            Long: +station.Long,
        }));
        // console.log('Stations loaded:', stations.length);
    } catch (error) {
        console.error('Error loading station data:', error);
    }
}

async function loadStationDemand() {
    try {
        const csvUrl = 'https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv';
        const data = await d3.csv(csvUrl);

        trips = data.map(trip => {
            return {
                id: trip.ride_id,
                name: trip.NAME,
                started_at: new Date(trip.started_at),
                ended_at: new Date(trip.ended_at),
                start_station_id: trip.start_station_id,
                end_station_id: trip.end_station_id
            };
        });

        // console.log('Trips loaded:', trips.length);
    } catch (error) {
        console.error('Error loading station data:', error);
    }
}

onMount(() => {
	initMap();
    loadStationData();
    loadStationDemand();

    const departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
    const arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

    stations = stations.map(station => {
        let id = station.id;
        station.arrivals = arrivals.get(id) ?? 0;
        station.departures = departures.get(id) ?? 0;
        station.totalTraffic = station.arrivals + station.departures;
        return station;
    });
});

function getCoords (station) {
	let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
	let {x, y} = map.project(point);
	return {cx: x, cy: y};
}

$: radiusScale = d3.scaleSqrt()
	.domain([0, d3.max(stations, d => d.totalTraffic) || 0])
	.range([0, 25]);



</script>

<h1>Welcome to SvelteKit</h1>
<p>Visit <a href="https://kit.svelte.dev">kit.svelte.dev</a> to read the documentation</p>

<div id="map">

    <svg>
        {#key mapViewChanged}
            {#each stations as station}
                <circle { ...getCoords(station) } r="3" fill="steelblue" />
            {/each}
        {/key}
    </svg>


</div>

<style>
@import url("$lib/global.css");
</style>
