<template>
  <div style="height: 550px">
    <div
      ref="mapContainer"
      style="height: 550px; width: 100%; position: relative; z-index: 1"
    ></div>
  </div>
</template>

<script>
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet.markercluster/dist/MarkerCluster.css";
import "leaflet.markercluster/dist/MarkerCluster.Default.css";
import Papa from "papaparse";
import "leaflet.markercluster";

export default {
  name: "MyMap",
  mounted() {
    if (!this.$refs.mapContainer) return;

    this.map = L.map(this.$refs.mapContainer).setView(
      [12.8797, 121.774], // Center of the Philippines
      6
    );

    L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
      maxZoom: 19,
      attribution:
        '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    }).addTo(this.map);

    this.loadMarkers();
  },
  beforeUnmount() {
    if (this.map) {
      this.map.remove();
    }
  },
  methods: {
    async loadMarkers() {
      try {
        const response = await fetch("/src/data/sites_07172024.csv");
        const csvData = await response.text();
        Papa.parse(csvData, {
          header: true,
          complete: (results) => {
            const markers = L.markerClusterGroup();

            results.data.forEach((site) => {
              const latitude = parseFloat(site.latitude);
              const longitude = parseFloat(site.longitude);

              // Check for valid latitude and longitude
              if (
                !isNaN(latitude) &&
                !isNaN(longitude) &&
                latitude !== 0 &&
                longitude !== 0
              ) {
                const marker = L.marker([latitude, longitude]).bindPopup(
                  `<b>Site ID:</b> ${site.site_id}<br><b>Province:</b> ${site.province}`
                );

                markers.addLayer(marker);
              }
            });

            this.map.addLayer(markers);
          },
        });
      } catch (error) {
        console.error("Error loading CSV data:", error);
      }
    },
  },
};
</script>

<style>
/* Optional: Import Leaflet CSS here as well if not using a global import */
/*@import "leaflet/dist/leaflet.css";*/
</style>
