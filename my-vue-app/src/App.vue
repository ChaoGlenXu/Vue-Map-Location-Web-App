<template>
  <div id="app">
    <h1>Location Finder</h1>

    <!-- Display time zone and local time -->
    <div>
      <p>Time Zone: {{ timeZone }}</p>
      <p>Local Time: {{ localTime }}</p>
    </div>

    <!-- Current location button -->
    <button @click="getCurrentLocation">Get Current Location</button>
    
    <!-- Search Module -->
    <input v-model="searchedLocation" @keyup.enter="searchLocation" placeholder="Search for a location" />
    <button @click="searchLocation">Search</button>
    
    <!-- Map display (here uses Google Maps service) -->
    <div id="map" style="width: 100%; height: 400px;"></div>

    <table>
      <thead>
        <tr>
          <th><button @click="deleteSelected">Delete</button></th>
          <th>Location</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="location in displayedLocations" :key="location.name">
          <td><input type="checkbox" v-model="location.selected"></td>
          <td> {{location.name}} </td>
        </tr>
      </tbody>
    </table>

    <!-- Pagination Controls -->
    <button @click="prevPage" :disabled="currentPage <= 0">Previous</button>
    <button @click="nextPage()" :disabled="currentPage >= maxPages - 1">Next</button>
  </div>
</template>


<script>
import axios from 'axios';

/*global google*/
export default {
  name: 'app',
  data() {
    return {
      searchedLocation: '',
      map: null,
      markers: [],  // Store markers to manage them
      locations: [],     // All searched locations.
      currentPage: 0,    // Current page for pagination.
      locationsPerPage: 10,

      //for displaying the time Zone and local Time
      timeZone: '',
      localTime: '',
    };
  },
  computed: {
    displayedLocations() {
      const start = this.currentPage * this.locationsPerPage;
      const end = start + this.locationsPerPage;
      return this.locations.slice(start, end);
    },
    maxPages() {
      return Math.ceil(this.locations.length / this.locationsPerPage);
    }
  },
  methods: {
    getCurrentLocation() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(position => {
          const location = {
            lat: position.coords.latitude,
            lng: position.coords.longitude
          };
          this.setMarker(location);
        });
      } else {
        alert("Geolocation is not supported by this browser.");
      }
    },
    searchLocation() {
      // Use Google Maps Places API (geocoding service)
      const geocoder = new google.maps.Geocoder();
      console.log(this.searchedLocation)
      geocoder.geocode({ 'address': this.searchedLocation }, (results, status) => {
        if (status == 'OK') {
          const location = results[0].geometry.location;
          this.setMarker({ lat: location.lat(), lng: location.lng() });

          this.fetchTimeZoneAndLocalTime(location.lat(), location.lng());

          // Add the location to the locations array
          this.locations.push({
          name: this.searchedLocation,
          marker: this.markers.at(-1),   // Storing reference to the marker.
          selected: false
          });

        } else {
          alert('Geocode was not successful for the following reason: ' + status);
        }
      });

      // this.locations.push({
      //   name: this.searchedLocation,
      //   marker: this.markers.at(-1),   // Storing reference to the marker.
      //   selected: false
      // });
      console.log(this.markers.at(-1))
    },
    setMarker(location) {
      // If map is not initialized, initialize it
      if (!this.map) {
        this.map = new google.maps.Map(document.getElementById('map'), {
          center: location,
          zoom: 8
        });
      }
      // Create a marker and add it to the map and markers array
      const marker = new google.maps.Marker({
        position: location,
        map: this.map
      });
      this.markers.push(marker);
      // Set map center to the new location
      this.map.setCenter(location);
    },
    deleteSelected() {
      for (let i = this.locations.length - 1; i >= 0; i--) {
        if (this.locations[i].selected) {
          this.locations[i].marker.setMap(null);  
          this.locations[i].marker.setVisible(false);
          console.log(this.locations[i].name)
          console.log(this.locations[i].marker == this.markers.at(-1))
          this.locations.splice(i, 1);

        }
      }
    },
    nextPage() {
      if (this.currentPage < this.maxPages - 1) {
        this.currentPage++;
      }
    },
    prevPage() {
      if (this.currentPage > 0) {
        this.currentPage--;
      }
    },


    async fetchTimeZoneAndLocalTime(latitude, longitude) {
      try {
        // Make an HTTP request to the Google Maps Time Zone API
        const apiKey = 'AIzaSyC3J3fVimM1wMWuDAhGOV0mE28PyzPbbd8';
        const timestamp = Math.floor(Date.now() / 1000);
        const response = await axios.get(
          `https://maps.googleapis.com/maps/api/timezone/json?location=${latitude},${longitude}&timestamp=${timestamp}&key=${apiKey}`
        );

        const { timeZoneName, timeZoneId } = response.data;

        // Get the current timestamp
        const currentTimestamp = timestamp;

        // Use Intl.DateTimeFormat to format the local time based on the time zone
        const localTime = new Intl.DateTimeFormat('en-US', {
          timeZone: timeZoneId,
          hour12: true,
          hour: 'numeric',
          minute: 'numeric',
          second: 'numeric',
          timeZoneName: 'short', // Include the time zone abbreviation
        }).format(currentTimestamp * 1000); // Convert to milliseconds

        // Update the data properties with the obtained information
        this.timeZone = `${timeZoneName} (${timeZoneId})`;
        this.localTime = localTime;
      } catch (error) {
        console.error('Error fetching time zone and local time:', error);
      }
    },


  }
};
</script>


<style>
#app {
  font-family: Arial, sans-serif;
  text-align: center;
  margin-top: 40px;
}
</style>

