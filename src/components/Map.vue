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
        const response = await fetch("/src/data/sites_07172024 - Copy.csv");
        const csvData = await response.text();
        Papa.parse(csvData, {
          header: true,
          complete: (results) => {
            const regions = {};

            // Define custom icons
            const softestYellowIcon = L.icon({
              iconUrl: "/src/assets/softestyellow-pin.png",
              iconSize: [25, 41],
              iconAnchor: [12, 41],
              popupAnchor: [1, -34],
            });

            const yellowIcon = L.icon({
              iconUrl: "/src/assets/oneday-yellow-pin.png",
              iconSize: [25, 41],
              iconAnchor: [12, 41],
              popupAnchor: [1, -34],
            });

            const orangeIcon = L.icon({
              iconUrl: "/src/assets/orange-pin.png",
              iconSize: [25, 41],
              iconAnchor: [12, 41],
              popupAnchor: [1, -34],
            });

            const redIcon = L.icon({
              iconUrl: "/src/assets/red-pin2.png",
              iconSize: [25, 41],
              iconAnchor: [12, 41],
              popupAnchor: [1, -34],
            });

            const darkRedIcon = L.icon({
              iconUrl: "/src/assets/darkred-pin.png",
              iconSize: [25, 41],
              iconAnchor: [12, 41],
              popupAnchor: [1, -34],
            });

            function parseDuration(duration) {
              if (!duration || typeof duration !== "string") {
                return 0;
              }

              const parts = duration.split(":");
              if (parts.length < 2) {
                return 0;
              }

              const hours = parseFloat(parts[0]) || 0;
              const minutes = parseFloat(parts[1]) || 0;

              return (hours * 60 + minutes) / 1440;
            }

            results.data.forEach((site) => {
              const region = site.region;
              const province = site.province;
              const latitude = parseFloat(site.latitude);
              const longitude = parseFloat(site.longitude);
              const downDuration = site.down_duration;

              const durationInDays = parseDuration(downDuration);

              let markerIcon;
              if (durationInDays < 1) {
                markerIcon = yellowIcon;
              } else if (durationInDays >= 1 && durationInDays <= 2) {
                markerIcon = orangeIcon;
              } else if (durationInDays >= 3 && durationInDays <= 6) {
                markerIcon = redIcon;
              } else if (durationInDays > 7) {
                markerIcon = darkRedIcon;
              } else {
                markerIcon = softestYellowIcon;
              }

              // Check for valid latitude and longitude
              if (
                !isNaN(latitude) &&
                !isNaN(longitude) &&
                latitude !== 0 &&
                longitude !== 0
              ) {
                if (!regions[region]) {
                  regions[region] = {
                    markers: [],
                    counts: {
                      yellow: 0,
                      orange: 0,
                      red: 0,
                      darkRed: 0,
                      softYellow: 0,
                    },
                  };
                }
                regions[region].markers.push({
                  latitude,
                  longitude,
                  site_id: site.site_id,
                  icon: markerIcon,
                  downDuration: durationInDays,
                  province: province,
                  region: region, // Ensure region is included
                });

                // Update counts
                if (durationInDays < 1) {
                  regions[region].counts.yellow++;
                } else if (durationInDays >= 1 && durationInDays <= 2) {
                  regions[region].counts.orange++;
                } else if (durationInDays >= 3 && durationInDays <= 6) {
                  regions[region].counts.red++;
                } else if (durationInDays > 7) {
                  regions[region].counts.darkRed++;
                } else {
                  regions[region].counts.softYellow++;
                }
              }
            });

            function getClusterClass(counts) {
              if (
                counts.yellow &&
                counts.orange === 0 &&
                counts.red === 0 &&
                counts.darkRed === 0
              ) {
                return "custom-cluster-yellow";
              } else if (
                counts.orange &&
                counts.red === 0 &&
                counts.darkRed === 0
              ) {
                return "custom-cluster-orange";
              } else if (counts.red && counts.darkRed === 0) {
                return "custom-cluster-red";
              } else if (counts.darkRed) {
                return "custom-cluster-dark-red";
              } else {
                return "custom-cluster-soft-yellow";
              }
            }

            function updatePopupContent(cluster) {
              const markers = cluster.getAllChildMarkers();
              const counts = {
                yellow: 0,
                orange: 0,
                red: 0,
                darkRed: 0,
                softYellow: 0,
              };

              const regionCounts = {};

              markers.forEach((marker) => {
                const downDuration = marker.options.downDuration;
                const region = marker.options.region;

                if (!regionCounts[region]) {
                  regionCounts[region] = {
                    yellow: 0,
                    orange: 0,
                    red: 0,
                    darkRed: 0,
                    softYellow: 0,
                  };
                }

                if (downDuration < 1) {
                  regionCounts[region].yellow++;
                } else if (downDuration >= 1 && downDuration <= 2) {
                  regionCounts[region].orange++;
                } else if (downDuration >= 3 && downDuration <= 6) {
                  regionCounts[region].red++;
                } else if (downDuration > 7) {
                  regionCounts[region].darkRed++;
                } else {
                  regionCounts[region].softYellow++;
                }
              });

              return Object.entries(regionCounts)
                .map(
                  ([regionName, counts]) => `
                  <b>Region:</b> ${regionName}<br>
                  <b>Down Durations:</b><br>
                  <ul>
                    <li><b>Less than 1 day:</b> ${counts.yellow || 0}<br></li>
                    <li><b>1 to 2 days:</b> ${counts.orange || 0}<br></li>
                    <li><b>3 to 6 days:</b> ${counts.red || 0}<br></li>
                    <li><b>More than 7 days:</b> ${counts.darkRed || 0}<br></li>
                    <li><b>Other:</b> ${counts.softYellow || 0}</li>
                  </ul>
                `
                )
                .join("<br>");
            }

            Object.keys(regions).forEach((region) => {
              const regionCluster = L.markerClusterGroup({
                iconCreateFunction: (cluster) => {
                  const counts = cluster.getAllChildMarkers().reduce(
                    (acc, marker) => {
                      const downDuration = marker.options.downDuration;
                      if (downDuration < 1) acc.yellow++;
                      else if (downDuration >= 1 && downDuration <= 2)
                        acc.orange++;
                      else if (downDuration >= 3 && downDuration <= 6)
                        acc.red++;
                      else if (downDuration > 7) acc.darkRed++;
                      else acc.softYellow++;
                      return acc;
                    },
                    {
                      yellow: 0,
                      orange: 0,
                      red: 0,
                      darkRed: 0,
                      softYellow: 0,
                    }
                  );

                  return L.divIcon({
                    html: `<div class=" custom-cluster-icon2 ${getClusterClass(
                      counts
                    )}">${cluster.getChildCount()}</div>`,
                    className: "custom-cluster-icon",
                    iconSize: [40, 40],
                  });
                },
              });

              const regionData = regions[region];

              regionData.markers.forEach((site) => {
                const marker = L.marker([site.latitude, site.longitude], {
                  icon: site.icon,
                  downDuration: site.downDuration,
                  region: site.region, // Ensure region is included
                }).bindPopup(
                  `<b>Site ID:</b> ${site.site_id}<br><b>Province:</b> ${site.province}<br><b>Region:</b> ${site.region}`
                );

                regionCluster.addLayer(marker);
              });

              this.map.addLayer(regionCluster);

              regionCluster.on("clustermouseover", (event) => {
                const cluster = event.layer;
                const popupContent = updatePopupContent(cluster);
                const popup = L.popup()
                  .setLatLng(cluster.getLatLng())
                  .setContent(popupContent);

                cluster.bindPopup(popup).openPopup();
              });

              regionCluster.on("clustermouseout", (event) => {
                event.layer.closePopup();
              });
            });
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
