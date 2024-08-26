<template>
  <ion-page>
    <ion-content :fullscreen="true">
      <!-- Search bar for entering a location -->
      <ion-list class="searchBar">
        <ion-item>
          <ion-input label="Ort:" placeholder="Hier suchen" v-model="searchLocation"></ion-input>
          <ion-button expand="block" id="search-button" @click="searchLocationOnMap">Suchen</ion-button>
        </ion-item>
      </ion-list>

      <!-- Container for the map -->    
      <div class="map-container">
        <l-map ref="map" :zoom="zoom" :center="mapCenter" @ready="onMapReady" :use-global-leaflet="false">
          <l-tile-layer 
            url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
            layer-type="base"
            name="OpenStreetMap"
          ></l-tile-layer>
          <l-marker :lat-lng="markerLatLng" :icon="searchIcon" @update:lat-lng="markerUpdate">
          </l-marker>
          <l-marker :lat-lng="markerCurrentPosition" :icon="userIcon" @update:lat-lng="markerUpdateCurrentPosition">
          </l-marker>
        </l-map>
      </div>

      <!-- Button for location retrieval -->
      <div class="location-container">
      <ion-button class="buttonSize" expand="block" @click="getLocationOnMap">
        <img src="https://cdn-icons-png.flaticon.com/512/565/565949.png">
      </ion-button>
      </div>

      <!-- Button for adjusting the map view -->
      <div class="positions-container">
      <ion-button class="buttonSize" expand="block"  @click="adjustMapView">
        <img src="https://cdn-icons-png.flaticon.com/512/156/156979.png" >
      </ion-button>
      </div>

      <!-- Progress bar shown during loading -->
      <ion-progress-bar v-if="isLoading" type="indeterminate"></ion-progress-bar>
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import "leaflet/dist/leaflet.css";
import { LMap, LTileLayer, LMarker } from "@vue-leaflet/vue-leaflet";
import { IonContent, IonList, IonItem, IonLabel, IonInput, IonButton, IonProgressBar } from "@ionic/vue"
import { latLng, icon, Map } from "leaflet"
import { Geolocation } from '@capacitor/geolocation';

export default {
  components: {
    LMap,
    LTileLayer,
    LMarker,
    IonContent,
    IonItem,
    IonLabel,
    IonInput,
    IonButton,
    IonProgressBar
  },

  data() {
    return {
      // Initial map and marker settings
      zoom: 2,
      mapCenter: latLng(47.41322, -1.219482),
      markerLatLng: latLng(47.41322, -1.219482),
      markerCurrentPosition: latLng(47.41322, -1.219482),
      searchLocation: "",
      map: null as Map | null,
      isLoading: false,
      storageData: {
        searchPos: {lat: 0, lon: 0},
        currentPos: {lat: 0, lon: 0},
      },
      // User location icon
      userIcon: icon({
        iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon.png', // Benutzerdefiniertes Bild für den Benutzerstandort
        iconSize: [30, 42],
        iconAnchor: [15, 42],
        popupAnchor: [0, -42],
        shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
        shadowSize: [42, 42],
        shadowAnchor: [15, 42],
      }),
      // Search location icon
      searchIcon: icon({
        iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-orange.png ', // Benutzerdefiniertes Bild für die Adresssuche
        iconSize: [30, 42],
        iconAnchor: [15, 42],
        popupAnchor: [0, -42],
        shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
        shadowSize: [42, 42],
        shadowAnchor: [15, 42],
      }),
    };
  },
  mounted() {
    // Load saved position and map settings from local storage
    let storage = localStorage.getItem("storagePosData");
    let mapSettings = localStorage.getItem("mapSettings");

    if (storage !== null) {
      this.storageData = JSON.parse(storage);

      // Update markers based on saved data
      this.markerUpdate(latLng(this.storageData.searchPos.lat, this.storageData.searchPos.lon));
      this.markerUpdateCurrentPosition(latLng(this.storageData.currentPos.lat, this.storageData.currentPos.lon));
    }
    if (mapSettings !== null) {
      const savedSettings = JSON.parse(mapSettings);
      // Update map center and zoom level based on saved settings
      this.mapCenter = latLng(savedSettings.center.lat, savedSettings.center.lng);
      this.zoom = savedSettings.zoom;
    }
  },
  methods: {
    // Initialize the map when it is ready
    onMapReady(mapInstance: Map) {
      this.map = mapInstance;
      // Only adjust the map if saved positions are available
      let storage = localStorage.getItem("storagePosData");
      let mapSettings = localStorage.getItem("mapSettings");
      if (mapSettings !== null) {
        const savedSettings = JSON.parse(mapSettings);
        // Set the map view to the saved position and zoom level
        this.map.setView(latLng(savedSettings.center.lat, savedSettings.center.lng), savedSettings.zoom);
      } 
      // Add a listener to save map settings when the map is moved
      this.map.on('moveend', this.saveMapSettings);
    },

  // Method to search for a location on the map
  searchLocationOnMap() {
    let places = [];
    let searchKey = this.searchLocation
    if (searchKey === '') {
        places = [];
      } else if (searchKey.length > 2) {
        let url = 'https://nominatim.openstreetmap.org/search?format=json&q=' + searchKey;
        this.isLoading = true
        fetch(url, {
            method: "GET"
          }).then((response) => {
            return response.json();
          }).then((data) => {
            // Use the coordinates from the search result
            console.log(data[0])
            let coords = {
            lat: data[0].lat,
            lon: data[0].lon
            }
            console.log('Koordinaten:',coords)
            this.map.setView(latLng(coords.lat, coords.lon), 15);
            this.storageData.searchPos = coords
            console.log(this.storageData)
            // Save the search position in local storage
            localStorage.setItem("storagePosData", JSON.stringify(this.storageData));
            this.markerUpdate(latLng(this.storageData.searchPos.lat, this.storageData.searchPos.lon))
            this.isLoading = false;
            console.log('Marker für die Suche:', this.markerLatLng);
            console.log('Marker für aktuelle Position:', this.markerCurrentPosition);
            console.log('gespeicherte Daten', this.storageData);
          })
    }
    },
    
  // Method to check geolocation permissions
  checkPermissions() {
    return Geolocation.checkPermissions();
  },

  // Method to request geolocation permissions
  requestPermissions(permissions) {
    return Geolocation.requestPermissions(permissions);
  },

// Method to retrieve the current location and update the map
async getLocationOnMap() {
  try {
    // Always request location permissions from the user
    const requestedPermissions = await Geolocation.requestPermissions({
      android: {
        accessFineLocation: true,
        accessCoarseLocation: true
      }
    });

    // Check the new permission status
    if (requestedPermissions.location === 'granted') {
      // If permission is granted after the request, retrieve the current location
      await this.retrieveCurrentLocation();
    } else {
      // Handle the case where the user denies the location access request
      console.error("Location access denied by user.");
      alert("Location access is required to display your current position on the map.");
    }
  } catch (error) {
    console.error("Error requesting location permissions: ", error);
    alert("An error occurred while accessing location services.");
  }
},

// Helper method to actually retrieve and handle the current location
async retrieveCurrentLocation() {
  this.isLoading = true;

  try {
    const position = await Geolocation.getCurrentPosition();
    const currentPosition = {
      lat: position.coords.latitude,
      lon: position.coords.longitude
    };

    console.log('Current position:', currentPosition);
    this.markerUpdateCurrentPosition(latLng(currentPosition.lat, currentPosition.lon));
    this.map.setView(latLng(currentPosition.lat, currentPosition.lon), 15);
    this.storageData.currentPos = currentPosition;

    // Save the current position in local storage
    localStorage.setItem("storagePosData", JSON.stringify(this.storageData));
    
    console.log('Marker for search:', this.markerLatLng);
    console.log('Marker for current position:', this.markerCurrentPosition);
    console.log('Saved data', this.storageData);
  } catch (error) {
    console.error("Error retrieving current position: ", error);
    alert("Unable to retrieve your current location.");
  } finally {
    this.isLoading = false;
  }
},

  // Method to adjust the map view to fit all markers
  adjustMapView() {
    if (this.map) {
      this.map.fitBounds([[this.storageData.currentPos.lat, this.storageData.currentPos.lon],
      [this.storageData.searchPos.lat, this.storageData.searchPos.lon]],
      {padding: [100, 100]});
    }
    },

  // Save map settings when the map is moved
  saveMapSettings() {
      if (this.map) {
        const mapSettings = {
          center: this.map.getCenter(),
          zoom: this.map.getZoom()
        };
        localStorage.setItem("mapSettings", JSON.stringify(mapSettings));
      }
    },

  // Update the marker for the search location
  markerUpdate(markerLatLng){
    this.markerLatLng = markerLatLng;
  },

  // Update the marker for the current position
  markerUpdateCurrentPosition(markerCurrentPosition){
    this.markerCurrentPosition = markerCurrentPosition;
  }

  },
  };
</script>

<style scoped>

.searchBar {
  position: fixed;
  margin-top: 10px;
  margin-bottom: 10px;
  z-index: 1;
  margin-left: 10px;
  margin-bottom: 10px;
  display: flex; 
  align-items: center; 
  justify-content: center;
  border-radius: 10px;
}

.header {
  font-size: 20px;
  font-weight: bold;
  color: rgb(80, 141, 255);
  text-align: center;
  margin-top: 10px;
  margin-bottom: 10px;
}
.buttonSize {
  width: 60px;
  height: 60px;
}

.map-container {
  z-index: 0;
  height: 100%;
  width: 100%;
  position: absolute;
}
.positions-container {
  position: fixed;
  z-index: 1;
  display: flex;
  float: right;
  bottom: 0;
  margin-left: 10px;
  margin-bottom: 10px;

}
.location-container {
  position: fixed;
  z-index: 1;
  display: flex;
  bottom: 0;
  right: 0;
  margin-right: 10px;
  margin-bottom: 10px;
}
#container {
  text-align: center;
  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

#container strong {
  font-size: 20px;
  line-height: 26px;
}

#container p {
  font-size: 16px;
  line-height: 22px;
  
  color: #8c8c8c;
  
  margin: 0;
}

#container a {
  text-decoration: none;
}
</style>