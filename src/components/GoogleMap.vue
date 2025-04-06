<script setup>
import { ref, onMounted, reactive, computed } from 'vue';
import { Loader } from '@googlemaps/js-api-loader';

// Google Maps API key (it should be stored in .env file in real project)
const API_KEY = "AIzaSyAyuxLXyDbnadIRQjrZAD8PdqDoKoi5ibw";

const mapDiv = ref(null);
const searchInput = ref(null);
const map = ref(null);
// markers are now reactive
const markers = reactive([]);
const polyline = ref(null);
const infoWindow = ref(null);
const searchBox = ref(null);
const directionsService = ref(null);
const directionsRenderer = ref(null);
const placesService = ref(null);
const streetViewPanorama = ref(null);
let googleInstance = null; // Let's keep Google instance globally

// Map layers
const trafficLayer = ref(null);
const transitLayer = ref(null);
const bicyclingLayer = ref(null);

const mapInfo = reactive({
  mapType: 'roadmap',
  layer: {
    traffic: false,
    transit: false,
    bicycling: false,
  },
  mapCenter: { lat: 41.015137, lng: 28.979530 }, // Istanbul
  activeMarker: null,
  markers: [],
  distance: 0,
  placeDetails: null,
  error: null,
  isLoading: false,
  searchQuery: '',
  routeMode: 'DRIVING',
  routeOptimized: false,
  routeInfo: null,
  nearbyPlaces: [],
  nearbyRadius: 500,
  nearbyType: 'restaurant',
  // Route preferences
  avoidHighways: false,
  avoidTolls: false,
  avoidFerries: false,
  showRouteDetails: false
});

// Available travel modes
const travelModes = [
  { value: 'DRIVING', label: 'Car', icon: '🚗' },
  { value: 'WALKING', label: 'Walk', icon: '🚶' },
  { value: 'BICYCLING', label: 'Bike', icon: '🚲' },
  { value: 'TRANSIT', label: 'Transit', icon: '🚌' }
];

// Place type options
const placeTypes = [
  { value: 'restaurant', label: 'Restaurants', icon: '🍽️' },
  { value: 'cafe', label: 'Cafes', icon: '☕' },
  { value: 'hotel', label: 'Hotels', icon: '🏨' },
  { value: 'shopping_mall', label: 'Malls', icon: '🛍️' },
  { value: 'tourist_attraction', label: 'Attractions', icon: '🏛️' },
  { value: 'gas_station', label: 'Gas Stations', icon: '⛽' },
  { value: 'hospital', label: 'Hospitals', icon: '🏥' },
  { value: 'pharmacy', label: 'Pharmacies', icon: '💊' },
  { value: 'bank', label: 'Banks', icon: '🏦' },
  { value: 'atm', label: 'ATMs', icon: '💸' }
];

// Formatted distance and duration for calculated values
const formattedDistance = computed(() => {
  if (!mapInfo.routeInfo || !mapInfo.routeInfo.distance) return "-";
  
  // Convert meters to a suitable unit
  const distance = mapInfo.routeInfo.distance;
  if (distance < 1000) {
    return `${distance} m`;
  } else {
    return `${(distance / 1000).toFixed(1)} km`;
  }
});

const formattedDuration = computed(() => {
  if (!mapInfo.routeInfo || !mapInfo.routeInfo.duration) return "-";
  
  // Convert seconds to a suitable unit
  const duration = mapInfo.routeInfo.duration;
  const hours = Math.floor(duration / 3600);
  const minutes = Math.floor((duration % 3600) / 60);
  
  if (hours > 0) {
    return `${hours} hr ${minutes} min`;
  } else {
    return `${minutes} min`;
  }
});

// Google Maps API loading control variable
let isLoaderInitialized = false;
let googleLoader = null;

onMounted(async () => {
  try {
    mapInfo.isLoading = true;
    
    // Load Google Maps API
    if (!googleLoader) {
      googleLoader = new Loader({
        apiKey: API_KEY,
        version: "weekly",
        libraries: ["places", "geometry", "elevation", "visualization"],
      });
      isLoaderInitialized = true;
    }

    const google = await googleLoader.load();
    googleInstance = google; // Keep Google instance
    
    // Map creation
    map.value = new google.maps.Map(mapDiv.value, {
      center: { lat: 41.0082, lng: 28.9784 }, // Istanbul coordinates
      zoom: 10,
      mapTypeId: google.maps.MapTypeId.ROADMAP,
      mapTypeControl: true,
      fullscreenControl: true,
      streetViewControl: true,
    });

    // Service and layers initialization
    initServicesAndLayers(google);

    // InfoWindow creation
    infoWindow.value = new google.maps.InfoWindow();

    // Map click event listener
    map.value.addListener("click", (event) => {
      if (!mapInfo.showStreetView) {
        addMarker(event.latLng, google);
      }
    });
    
    // InfoWindow closed event listener
    infoWindow.value.addListener("closeclick", () => {
      // Actions to be taken when InfoWindow is closed
    });
    
    // Search box creation
    initSearchBox(google);
    
    mapInfo.isLoading = false;
  } catch (error) {
    mapInfo.error = "Error loading map: " + error.message;
    mapInfo.isLoading = false;
    console.error("Map loading error:", error);
  }
});

// Service and layers initialization
function initServicesAndLayers(google) {
  // Directions service creation
  directionsService.value = new google.maps.DirectionsService();
  directionsRenderer.value = new google.maps.DirectionsRenderer({
    suppressMarkers: true, // We'll use our own markers
    polylineOptions: {
      strokeColor: '#4285F4',
      strokeWeight: 5,
      strokeOpacity: 0.8
    }
  });
  directionsRenderer.value.setMap(map.value);
  
  // Places servisi oluştur
  if (map.value) {
    placesService.value = new google.maps.places.PlacesService(map.value);
  }
  
  // Trafik, toplu taşıma ve bisiklet katmanlarını oluştur
  trafficLayer.value = new google.maps.TrafficLayer();
  transitLayer.value = new google.maps.TransitLayer();
  bicyclingLayer.value = new google.maps.BicyclingLayer();
}

// Arama kutusunu başlat
function initSearchBox(google) {
  // Arama kutusunu kontrol et
  if (!searchInput.value) {
    console.error("Search input reference not found");
    return;
  }
  
  try {
    // SearchBox oluştur
    searchBox.value = new google.maps.places.SearchBox(searchInput.value);
    
    // Arama kutusu haritaya sınırlansın
    map.value.addListener("bounds_changed", () => {
      searchBox.value.setBounds(map.value.getBounds());
    });
    
    // Arama sonuçları değiştiğinde
    searchBox.value.addListener("places_changed", () => {
      const places = searchBox.value.getPlaces();
      
      if (places.length === 0) {
        return;
      }
      
      // Aranan yerleri sakla
      mapInfo.searchResults = places;
      
      // İlk yeri seç ve haritaya git
      const place = places[0];
      if (!place.geometry || !place.geometry.location) {
        console.error("No geometry found for selected place");
        return;
      }
      
      // Get detailed place information
      getPlaceDetails(place.place_id, googleInstance);
      
      // Move map to searched location
      map.value.setCenter(place.geometry.location);
      
      // Mark the searched location
      const marker = new google.maps.Marker({
        map: map.value,
        position: place.geometry.location,
        title: place.name,
        animation: google.maps.Animation.DROP,
        icon: {
          url: "https://maps.google.com/mapfiles/ms/icons/blue-dot.png"
        }
      });
      
      // Show information window about the searched place
      const content = `
        <div>
          <h3 class="text-lg font-semibold">${place.name}</h3>
          <p class="text-gray-600">${place.formatted_address || ''}</p>
          <div class="flex mt-2 space-x-2">
            <button id="add-to-route" class="px-2 py-1 bg-blue-500 text-white rounded hover:bg-blue-600">Add to Route</button>
            <button id="show-street-view" class="px-2 py-1 bg-green-500 text-white rounded hover:bg-green-600">Street View</button>
            <button id="find-nearby" class="px-2 py-1 bg-purple-500 text-white rounded hover:bg-purple-600">Nearby Places</button>
          </div>
        </div>
      `;
      
      infoWindow.value.setContent(content);
      infoWindow.value.open(map.value, marker);
      
      // Add event listeners for buttons
      setTimeout(() => {
        // Add to Route button
        const addButton = document.getElementById("add-to-route");
        if (addButton) {
          addButton.addEventListener("click", () => {
            addMarker(place.geometry.location, googleInstance);
            infoWindow.value.close();
          });
        }
        
        // Street View button
        const streetViewButton = document.getElementById("show-street-view");
        if (streetViewButton) {
          streetViewButton.addEventListener("click", () => {
            showStreetView(place.geometry.location, googleInstance);
            infoWindow.value.close();
          });
        }
        
        // Nearby Places button
        const nearbyButton = document.getElementById("find-nearby");
        if (nearbyButton) {
          nearbyButton.addEventListener("click", () => {
            findNearbyPlaces(place.geometry.location, googleInstance);
            infoWindow.value.close();
          });
        }
      }, 100);
    });
  } catch (error) {
    console.error("Error creating search box:", error);
  }
}

// Get detailed place information
function getPlaceDetails(placeId, google) {
  if (!placesService.value) return;
  
  placesService.value.getDetails(
    { placeId: placeId, fields: ['name', 'formatted_address', 'photos', 'rating', 'website', 'formatted_phone_number', 'opening_hours', 'reviews', 'types', 'price_level'] },
    (place, status) => {
      if (status === google.maps.places.PlacesServiceStatus.OK) {
        mapInfo.placeDetails = place;
      }
    }
  );
}

// Show street view
function showStreetView(location, google) {
  mapInfo.showStreetView = true;
  
  // Create street view panorama
  streetViewPanorama.value = new google.maps.StreetViewPanorama(
    mapDiv.value,
    {
      position: location,
      pov: { heading: 0, pitch: 0 },
      zoom: 1,
      addressControl: true,
      linksControl: true,
    }
  );
  
  map.value.setStreetView(streetViewPanorama.value);
  
  // Add exit button for street view
  const exitButton = document.createElement('button');
  exitButton.className = 'absolute top-4 right-4 bg-white rounded-full p-2 shadow-lg border border-gray-300 hover:bg-gray-100 z-20';
  exitButton.innerHTML = '<span class="text-red-500 font-bold text-xl">✕</span>';
  exitButton.onclick = () => exitStreetView(google);
  
  mapDiv.value.appendChild(exitButton);
}

// Exit street view
function exitStreetView(google) {
  if (streetViewPanorama.value) {
    mapInfo.showStreetView = false;
    map.value.setStreetView(null);
    
    // Show map again
    const exitButton = mapDiv.value.querySelector('button');
    if (exitButton) {
      mapDiv.value.removeChild(exitButton);
    }
  }
}

// Find nearby places
function findNearbyPlaces(location, google) {
  if (!placesService.value) return;
  
  const request = {
    location: location,
    radius: mapInfo.nearbyRadius,
    type: mapInfo.nearbyType
  };
  
  placesService.value.nearbySearch(request, (results, status) => {
    if (status === google.maps.places.PlacesServiceStatus.OK) {
      mapInfo.nearbyPlaces = results;
      
      // Mark nearby places on the map
      results.forEach(place => {
        const marker = new google.maps.Marker({
          map: map.value,
          position: place.geometry.location,
          title: place.name,
          icon: {
            url: "https://maps.google.com/mapfiles/ms/icons/yellow-dot.png"
          }
        });
        
        // Show information window when marker is clicked
        marker.addListener('click', () => {
          const rating = place.rating ? `⭐ ${place.rating.toFixed(1)}` : '';
          const content = `
            <div>
              <h3 class="text-lg font-semibold">${place.name}</h3>
              <p class="text-sm text-gray-600">${place.vicinity || ''}</p>
              <p class="text-amber-500">${rating}</p>
              <button id="add-to-route" class="mt-2 px-2 py-1 bg-blue-500 text-white rounded hover:bg-blue-600">Add to Route</button>
            </div>
          `;
          
          infoWindow.value.setContent(content);
          infoWindow.value.open(map.value, marker);
          
          // Add event for Add to Route button
          setTimeout(() => {
            const addButton = document.getElementById("add-to-route");
            if (addButton) {
              addButton.addEventListener("click", () => {
                addMarker(place.geometry.location, google);
                infoWindow.value.close();
              });
            }
          }, 100);
        });
      });
    }
  });
}

/**
 * Calculate optimized route
 */
async function calculateOptimizedRoute(google) {
  if (!google || markers.length < 2) return;
  
  mapInfo.isLoading = true;
  
  try {
    // Create DirectionsService
    const directionsService = new google.maps.DirectionsService();
    
    // Set start and end points
    const origin = markers[0].position;
    const destination = markers[markers.length - 1].position;
    
    // Ara noktaları oluştur (başlangıç ve bitiş hariç)
    const waypoints = markers.slice(1, markers.length - 1).map(marker => ({
      location: marker.position,
      stopover: true
    }));
    
    // Seyahat modunu belirle
    let travelMode = google.maps.TravelMode[mapInfo.routeMode];
    
    // DirectionsRequest ayarlarını oluştur
    const request = {
      origin: origin,
      destination: destination,
      waypoints: waypoints,
      travelMode: travelMode,
      optimizeWaypoints: true,
      provideRouteAlternatives: true,
      avoidHighways: mapInfo.avoidHighways,
      avoidTolls: mapInfo.avoidTolls,
      avoidFerries: mapInfo.avoidFerries,
      drivingOptions: mapInfo.routeMode === 'DRIVING' ? {
        departureTime: new Date(), // Şu anki zaman
        trafficModel: google.maps.TrafficModel.BEST_GUESS
      } : undefined,
      transitOptions: mapInfo.routeMode === 'TRANSIT' ? {
        departureTime: new Date(),
        modes: [google.maps.TransitMode.BUS, google.maps.TransitMode.RAIL],
        routingPreference: google.maps.TransitRoutePreference.FEWER_TRANSFERS
      } : undefined
    };
    
    // Rotayı hesapla
    const result = await new Promise((resolve, reject) => {
      directionsService.route(request, (response, status) => {
        if (status === google.maps.DirectionsStatus.OK) {
          resolve(response);
        } else {
          reject(new Error(`Route calculation failed: ${status}`));
        }
      });
    });
    
    // DirectionsRenderer oluştur ve haritaya bağla
    if (!directionsRenderer.value) {
      directionsRenderer.value = new google.maps.DirectionsRenderer({
        map: map.value,
        suppressMarkers: true, // Varolan işaretçileri kullanmak için
        preserveViewport: true
      });
    }
    
    // Hesaplanan rotayı göster
    directionsRenderer.value.setDirections(result);
    
    // Optimize edilmiş rota sırasını al
    if (result.routes && result.routes.length > 0) {
      const route = result.routes[0];
      
      // Rota bilgilerini kaydet
      const routeInfo = {
        distance: route.legs.reduce((total, leg) => total + leg.distance.value, 0),
        duration: route.legs.reduce((total, leg) => total + leg.duration.value, 0),
        durationInTraffic: route.legs.some(leg => leg.duration_in_traffic) ? 
          route.legs.reduce((total, leg) => total + (leg.duration_in_traffic ? leg.duration_in_traffic.text : leg.duration.value), 0) : null,
        routes: result.routes.map((r, index) => ({
          index: index,
          name: index === 0 ? 'Recommended Route' : `Alternative ${index}`,
          distance: r.legs.reduce((total, leg) => total + leg.distance.value, 0),
          duration: r.legs.reduce((total, leg) => total + leg.duration.value, 0),
          summary: r.summary,
          isSelected: index === 0,
          bounds: r.bounds
        })),
        steps: route.legs.flatMap(leg => 
          leg.steps.map(step => ({
            instructions: step.instructions,
            distance: step.distance.text,
            duration: step.duration.text,
            maneuver: step.maneuver
          }))
        )
      };
      
      mapInfo.routeInfo = routeInfo;
      
      // Optimizasyon sonucunda nokta sırası değiştiyse markerleri güncelle
      if (route.waypoint_order && route.waypoint_order.length > 0) {
        const newOrder = [0]; // Başlangıç noktası her zaman ilk sırada
        
        // Orta noktaların yeni sırasını ekle
        for (const index of route.waypoint_order) {
          newOrder.push(index + 1); // waypoint_order 0'dan başlar, bizim dizimiz 1'den başlıyor
        }
        
        newOrder.push(markers.length - 1); // Bitiş noktası her zaman son sırada
        
        // Mevcut sıralama ile yeni sıralamayı karşılaştır
        const currentOrder = markers.map((_, index) => index);
        
        if (!arraysEqual(currentOrder, newOrder)) {
          // Sıralama değiştiyse kullanıcıya bildir
          alert("Route optimized! Marker order has been changed.");
          
          // Markerların sırasını değiştir
          const newMarkers = newOrder.map(index => markers[index]);
          markers.splice(0, markers.length, ...newMarkers);
        }
      }
    }
  } catch (error) {
    console.error("Route calculation error:", error);
    alert(`Route calculation failed: ${error.message}`);
  } finally {
    mapInfo.isLoading = false;
  }
}

/**
 * Alternatif rotayı göster
 */
function showAlternativeRoute(routeIndex, google) {
  if (!directionsRenderer.value || !google || !mapInfo.routeInfo || !mapInfo.routeInfo.routes) return;
  
  // Tüm rotaları deselect yap
  mapInfo.routeInfo.routes.forEach(route => {
    route.isSelected = route.index === routeIndex;
  });
  
  // Seçilen rotanın indeksini belirleyelim
  directionsRenderer.value.setRouteIndex(routeIndex);
  
  // Seçilen rotanın adımlarını güncelle
  if (directionsRenderer.value.getDirections() && directionsRenderer.value.getDirections().routes) {
    const selectedRoute = directionsRenderer.value.getDirections().routes[routeIndex];
    if (selectedRoute) {
      mapInfo.routeInfo.steps = selectedRoute.legs.flatMap(leg => 
        leg.steps.map(step => ({
          instructions: step.instructions,
          distance: step.distance.text,
          duration: step.duration.text,
          maneuver: step.maneuver || ''
        }))
      );
      
      // Mesafe ve süre bilgilerini güncelle
      mapInfo.routeInfo.distance = selectedRoute.legs.reduce((total, leg) => total + leg.distance.value, 0);
      mapInfo.routeInfo.duration = selectedRoute.legs.reduce((total, leg) => total + leg.duration.value, 0);
    }
  }
}

/**
 * İki dizinin eşit olup olmadığını kontrol et
 */
function arraysEqual(a, b) {
  if (a.length !== b.length) return false;
  for (let i = 0; i < a.length; i++) {
    if (a[i] !== b[i]) return false;
  }
  return true;
}

// Haritayı temizle ve yeniden başlat (acil durumlar için)
function resetMap() {
  // Doğrudan sayfayı yenile
  window.location.reload();
}

// Manevra ikonlarını döndür
function getManeuverIcon(maneuver) {
  switch (maneuver) {
    case 'turn-right': return '➡️';
    case 'turn-left': return '⬅️';
    case 'turn-slight-right': return '↗️';
    case 'turn-slight-left': return '↖️';
    case 'turn-sharp-right': return '↘️';
    case 'turn-sharp-left': return '↙️';
    case 'keep-right': return '➡️';
    case 'keep-left': return '⬅️';
    case 'uturn-right': return '⤵️';
    case 'uturn-left': return '⤴️';
    case 'ramp-right': return '↘️';
    case 'ramp-left': return '↙️';
    case 'merge': return '↔️';
    case 'fork-right': return '↱';
    case 'fork-left': return '↰';
    case 'ferry': return '⚓';
    case 'ferry-train': return '⚓';
    case 'roundabout-right': return '⟳';
    case 'roundabout-left': return '⟲';
    default: return '↑';
  }
}

// Harita katmanlarını değiştirme fonksiyonları
function toggleTrafficLayer(google) {
  mapInfo.layer.traffic = !mapInfo.layer.traffic;
  
  if (mapInfo.layer.traffic) {
    if (!trafficLayer.value) {
      trafficLayer.value = new google.maps.TrafficLayer();
    }
    trafficLayer.value.setMap(map.value);
  } else if (trafficLayer.value) {
    trafficLayer.value.setMap(null);
  }
}

function toggleTransitLayer(google) {
  mapInfo.layer.transit = !mapInfo.layer.transit;
  
  if (mapInfo.layer.transit) {
    if (!transitLayer.value) {
      transitLayer.value = new google.maps.TransitLayer();
    }
    transitLayer.value.setMap(map.value);
  } else if (transitLayer.value) {
    transitLayer.value.setMap(null);
  }
}

function toggleBicyclingLayer(google) {
  mapInfo.layer.bicycling = !mapInfo.layer.bicycling;
  
  if (mapInfo.layer.bicycling) {
    if (!bicyclingLayer.value) {
      bicyclingLayer.value = new google.maps.BicyclingLayer();
    }
    bicyclingLayer.value.setMap(map.value);
  } else if (bicyclingLayer.value) {
    bicyclingLayer.value.setMap(null);
  }
}

// Haritaya işaretçi ekle
function addMarker(location, google) {
  if (!map.value || !google) return;
  
  const markerId = Date.now();
  
  // Yeni işaretçi oluştur
  const markerObj = new google.maps.Marker({
    position: location,
    map: map.value,
    animation: google.maps.Animation.DROP,
    draggable: true,
    title: `Location #${markers.length + 1}`
  });
  
  // Marker'a özel ID ata
  markerObj.set("markerId", markerId);
  
  // İşaretçi bilgilerini sakla
  const markerInfo = {
    id: markerId,
    position: {
      lat: location.lat(),
      lng: location.lng()
    },
    marker: markerObj
  };
  
  // markers dizisine ekle
  markers.push(markerInfo);
  
  // İşaretçi sürüklendiğinde pozisyonu güncelle
  markerObj.addListener("dragend", () => {
    const newPosition = markerObj.getPosition();
    
    // İşaretçi pozisyonunu güncelle
    const marker = markers.find(m => m.id === markerId);
    if (marker) {
      marker.position = {
        lat: newPosition.lat(),
        lng: newPosition.lng()
      };
      
      // Rotayı yeniden hesapla
      if (mapInfo.routeOptimized && markers.length >= 2) {
        calculateOptimizedRoute(google);
      }
    }
  });
  
  // İşaretçiye tıkladığında bilgi penceresi göster
  markerObj.addListener("click", () => {
    const index = markers.findIndex(m => m.id === markerId);
    const pointNumber = index + 1;
    
    const infoContent = `
      <div>
        <h3 class="text-lg font-semibold">Location #${pointNumber}</h3>
        <p>Lat: ${location.lat().toFixed(6)}</p>
        <p>Lng: ${location.lng().toFixed(6)}</p>
        <div class="flex space-x-2 mt-2">
          <button id="delete-marker" data-marker-id="${markerId}" class="px-2 py-1 bg-red-500 text-white rounded hover:bg-red-600">Remove</button>
          <button id="show-street-view-marker" class="px-2 py-1 bg-green-500 text-white rounded hover:bg-green-600">Street View</button>
        </div>
      </div>
    `;
    
    infoWindow.value.setContent(infoContent);
    infoWindow.value.open(map.value, markerObj);
    
    // Düğmeler için olay dinleyicileri ekle
    setTimeout(() => {
      // Silme düğmesi
      const deleteButton = document.getElementById("delete-marker");
      if (deleteButton) {
        deleteButton.addEventListener("click", () => {
          const idToDelete = parseInt(deleteButton.getAttribute("data-marker-id"));
          deleteMarker(idToDelete, google);
          infoWindow.value.close();
        });
      }
      
      // Sokak görünümü düğmesi
      const streetViewButton = document.getElementById("show-street-view-marker");
      if (streetViewButton) {
        streetViewButton.addEventListener("click", () => {
          showStreetView(location, google);
          infoWindow.value.close();
        });
      }
    }, 100);
  });
  
  // En az 2 işaretçi varsa, rota hesapla
  if (markers.length >= 2) {
    if (mapInfo.routeOptimized) {
      calculateOptimizedRoute(google);
    }
  }
  
  return markerInfo;
}

// İşaretçiyi sil
function deleteMarker(id, google) {
  // İşaretçiyi dizide bul
  const index = markers.findIndex(m => m.id === id);
  
  if (index !== -1) {
    const marker = markers[index];
    
    // Google Maps'ten kaldır
    if (marker.marker) {
      google.maps.event.clearInstanceListeners(marker.marker);
      marker.marker.setMap(null);
    }
    
    // Diziden kaldır
    markers.splice(index, 1);
    
    // Rotayı güncelle
    if (markers.length >= 2 && mapInfo.routeOptimized) {
      calculateOptimizedRoute(google);
    } else {
      // DirectionsRenderer'ı temizle
      if (directionsRenderer.value) {
        directionsRenderer.value.setMap(null);
        directionsRenderer.value.setDirections({routes: []});
      }
      
      mapInfo.routeInfo = null;
      mapInfo.routeOptimized = false;
    }
  }
}

// Tüm işaretçileri temizle
function clearAllMarkers(google) {
  // Tüm marker'ları haritadan kaldır
  markers.forEach(markerInfo => {
    if (markerInfo.marker) {
      google.maps.event.clearInstanceListeners(markerInfo.marker);
      markerInfo.marker.setMap(null);
    }
  });
  
  // Dizileri temizle
  markers.splice(0, markers.length);
  
  // DirectionsRenderer'ı temizle
  if (directionsRenderer.value) {
    directionsRenderer.value.setMap(null);
    directionsRenderer.value.setDirections({routes: []});
  }
  
  // Rota bilgisini sıfırla
  mapInfo.routeInfo = null;
  mapInfo.routeOptimized = false;
}

// Harita tipini değiştir
function changeMapType(type, google) {
  if (!map.value) return;
  
  switch (type) {
    case 'roadmap':
      map.value.setMapTypeId(google.maps.MapTypeId.ROADMAP);
      break;
    case 'satellite':
      map.value.setMapTypeId(google.maps.MapTypeId.SATELLITE);
      break;
    case 'hybrid':
      map.value.setMapTypeId(google.maps.MapTypeId.HYBRID);
      break;
    case 'terrain':
      map.value.setMapTypeId(google.maps.MapTypeId.TERRAIN);
      break;
  }
  
  mapInfo.mapType = type;
}

// Saniye cinsinden süreyi formatla
function formatDuration(seconds) {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  
  if (hours > 0) {
    return `${hours} hr ${minutes} min`;
  }
  return `${minutes} min`;
}

// Seyahat modu seçimi
function setTravelMode(mode, google) {
  mapInfo.routeMode = mode;
  if (markers.length >= 2) {
    calculateOptimizedRoute(google);
  }
}
</script>

<template>
  <div class="w-full h-full flex flex-col">
    <div v-if="mapInfo.isLoading" class="absolute inset-0 flex items-center justify-center bg-black bg-opacity-50 z-50">
      <div class="text-white text-xl">Loading Map...</div>
    </div>
    
    <div v-if="mapInfo.error" class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
      {{ mapInfo.error }}
    </div>

    <!-- iDIGITEK Maps Logo/Header -->
    <div class="w-full bg-indigo-700 text-white p-3 mb-4 rounded-lg shadow-md">
      <h1 class="text-2xl font-bold text-center">iDIGITEK Maps</h1>
    </div>

    <!-- Arama kutusu ve kontrol paneli -->
    <div class="w-full bg-white shadow-md rounded-lg p-4 mb-4">
      <div class="relative w-full">
        <div class="w-full">
          <input
            ref="searchInput"
            type="text"
            placeholder="Search location..."
            class="w-full px-4 py-3 rounded-lg border border-gray-300 shadow-sm text-gray-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
        </div>
      </div>
      
      <!-- Kontrol paneli -->
      <div class="flex flex-wrap mt-4 gap-2">
        <!-- Harita tipleri -->
        <div class="flex space-x-1 bg-gray-100 p-1 rounded-lg">
          <button 
            @click="changeMapType('roadmap', googleInstance)" 
            :class="[
              'px-3 py-1 rounded-md text-sm font-medium transition-colors', 
              mapInfo.mapType === 'roadmap' ? 'bg-blue-500 text-white' : 'bg-white text-gray-700 hover:bg-gray-200'
            ]"
          >
            Standard
          </button>
          <button 
            @click="changeMapType('satellite', googleInstance)" 
            :class="[
              'px-3 py-1 rounded-md text-sm font-medium transition-colors', 
              mapInfo.mapType === 'satellite' ? 'bg-blue-500 text-white' : 'bg-white text-gray-700 hover:bg-gray-200'
            ]"
          >
            Satellite
          </button>
          <button 
            @click="changeMapType('hybrid', googleInstance)" 
            :class="[
              'px-3 py-1 rounded-md text-sm font-medium transition-colors', 
              mapInfo.mapType === 'hybrid' ? 'bg-blue-500 text-white' : 'bg-white text-gray-700 hover:bg-gray-200'
            ]"
          >
            Hybrid
          </button>
          <button 
            @click="changeMapType('terrain', googleInstance)" 
            :class="[
              'px-3 py-1 rounded-md text-sm font-medium transition-colors', 
              mapInfo.mapType === 'terrain' ? 'bg-blue-500 text-white' : 'bg-white text-gray-700 hover:bg-gray-200'
            ]"
          >
            Terrain
          </button>
        </div>
        
        <!-- Harita katmanları -->
        <div class="flex space-x-1">
          <button 
            @click="toggleTrafficLayer(googleInstance)" 
            :class="[
              'px-3 py-1 rounded-md text-sm font-medium transition-colors',
              mapInfo.layer.traffic ? 'bg-red-500 text-white' : 'bg-white border border-gray-300 text-gray-700 hover:bg-gray-100'
            ]"
          >
            🚦 Traffic
          </button>
          <button 
            @click="toggleTransitLayer(googleInstance)" 
            :class="[
              'px-3 py-1 rounded-md text-sm font-medium transition-colors',
              mapInfo.layer.transit ? 'bg-green-500 text-white' : 'bg-white border border-gray-300 text-gray-700 hover:bg-gray-100'
            ]"
          >
            🚇 Transit
          </button>
          <button 
            @click="toggleBicyclingLayer(googleInstance)" 
            :class="[
              'px-3 py-1 rounded-md text-sm font-medium transition-colors',
              mapInfo.layer.bicycling ? 'bg-blue-500 text-white' : 'bg-white border border-gray-300 text-gray-700 hover:bg-gray-100'
            ]"
          >
            🚲 Biking
          </button>
        </div>
        
        <!-- Rota optimizasyonu -->
        <div v-if="markers.length >= 2" class="flex-grow text-right">
          <button 
            @click="calculateOptimizedRoute(googleInstance)" 
            class="px-3 py-1 bg-indigo-600 text-white text-sm rounded hover:bg-indigo-700 transition flex items-center"
            :disabled="mapInfo.isLoading"
          >
            <span v-if="mapInfo.isLoading" class="animate-spin mr-1">🔄</span>
            <span v-else class="mr-1">🔄</span>
            {{ mapInfo.isLoading ? 'Calculating...' : 'Optimize Route' }}
          </button>
        </div>
      </div>
      
      <!-- Seyahat modu seçimi -->
      <div v-if="markers.length >= 2" class="flex flex-wrap mt-3 space-x-2">
        <span class="text-gray-600 py-1">Mode:</span>
        <div class="flex space-x-1 bg-gray-100 p-1 rounded-lg">
          <button 
            v-for="mode in travelModes" 
            :key="mode.value"
            @click="setTravelMode(mode.value, googleInstance)" 
            :class="[
              'px-3 py-1 rounded-md text-sm font-medium transition-colors flex items-center', 
              mapInfo.routeMode === mode.value ? 'bg-blue-500 text-white' : 'bg-white text-gray-700 hover:bg-gray-200'
            ]"
          >
            <span class="mr-1">{{ mode.icon }}</span> {{ mode.label }}
          </button>
        </div>
      </div>
    </div>
    
    <!-- Harita -->
    <div class="h-[600px] w-full relative" ref="mapDiv">
      <!-- Sokak görünümü çıkış butonu otomatik eklenecek -->
    </div>
    
    <!-- Rota bilgileri bölümü -->
    <div class="p-4 bg-white shadow-md rounded-lg mt-4">
      <h2 class="text-xl font-bold mb-2">Route Information</h2>
      
      <div v-if="markers.length === 0" class="text-gray-500">
        Click on the map or search a location to start creating a route.
      </div>
      
      <div v-else>
        <div class="flex flex-wrap mb-4 gap-4">
          <div class="p-3 bg-blue-50 rounded-lg flex-grow">
            <p class="text-gray-600 text-sm">Points</p>
            <p class="text-xl font-bold">{{ markers.length }}</p>
          </div>
          
          <div class="p-3 bg-green-50 rounded-lg flex-grow">
            <p class="text-gray-600 text-sm">Distance</p>
            <p class="text-xl font-bold">{{ formattedDistance }}</p>
          </div>
          
          <div class="p-3 bg-amber-50 rounded-lg flex-grow">
            <p class="text-gray-600 text-sm">Duration</p>
            <p class="text-xl font-bold">{{ formattedDuration }}</p>
            <p v-if="mapInfo.routeInfo && mapInfo.routeInfo.durationInTraffic" class="text-xs text-orange-600 font-medium">
              In traffic: {{ mapInfo.routeInfo.durationInTraffic }}
            </p>
          </div>
        </div>
        
        <!-- Rota Optimizasyon Seçenekleri -->
        <div v-if="markers.length >= 2" class="mb-5 p-4 bg-gray-50 rounded-lg border border-gray-200">
          <div class="flex justify-between items-center mb-3">
            <h3 class="text-lg font-semibold">Route Options</h3>
            <button 
              @click="calculateOptimizedRoute(googleInstance)" 
              class="px-3 py-1 bg-indigo-600 text-white text-sm rounded hover:bg-indigo-700 transition flex items-center"
              :disabled="mapInfo.isLoading"
            >
              <span v-if="mapInfo.isLoading" class="animate-spin mr-1">🔄</span>
              <span v-else class="mr-1">🔄</span>
              {{ mapInfo.isLoading ? 'Calculating...' : 'Optimize Route' }}
            </button>
          </div>
          
          <div class="flex flex-wrap gap-x-6 gap-y-2 mb-2">
            <label class="flex items-center cursor-pointer">
              <input 
                type="checkbox" 
                v-model="mapInfo.avoidHighways" 
                class="w-4 h-4 text-blue-600 rounded border-gray-300 focus:ring-blue-500"
              >
              <span class="ml-2 text-sm text-gray-700">Avoid Highways</span>
            </label>
            
            <label class="flex items-center cursor-pointer">
              <input 
                type="checkbox" 
                v-model="mapInfo.avoidTolls" 
                class="w-4 h-4 text-blue-600 rounded border-gray-300 focus:ring-blue-500"
              >
              <span class="ml-2 text-sm text-gray-700">Avoid Tolls</span>
            </label>
            
            <label class="flex items-center cursor-pointer">
              <input 
                type="checkbox" 
                v-model="mapInfo.avoidFerries" 
                class="w-4 h-4 text-blue-600 rounded border-gray-300 focus:ring-blue-500"
              >
              <span class="ml-2 text-sm text-gray-700">Avoid Ferries</span>
            </label>
          </div>
        </div>
        
        <!-- Alternatif Rotalar -->
        <div v-if="mapInfo.routeInfo && mapInfo.routeInfo.routes && mapInfo.routeInfo.routes.length > 1" class="mb-4">
          <h3 class="text-lg font-semibold mb-2">Alternative Routes</h3>
          <div class="flex overflow-x-auto space-x-2 pb-2">
            <button 
              v-for="route in mapInfo.routeInfo.routes" 
              :key="route.index"
              @click="showAlternativeRoute(route.index, googleInstance)"
              :class="[
                'px-3 py-2 border rounded-lg min-w-max flex-shrink-0 whitespace-nowrap text-sm font-medium transition',
                route.isSelected 
                  ? 'bg-blue-100 border-blue-500 text-blue-700' 
                  : 'bg-white border-gray-300 text-gray-700 hover:bg-gray-50'
              ]"
            >
              <div class="flex flex-col">
                <span class="font-semibold">{{ route.name }}</span>
                <span class="text-xs">{{ route.distance }} • {{ route.duration }}</span>
                <span v-if="route.summary" class="text-xs text-gray-500 truncate max-w-[180px]">
                  {{ route.summary }}
                </span>
              </div>
            </button>
          </div>
        </div>
        
        <!-- Adım Adım Yol Tarifleri -->
        <div v-if="mapInfo.routeInfo && mapInfo.routeInfo.steps && mapInfo.routeInfo.steps.length > 0" class="mb-5">
          <div class="flex justify-between items-center">
            <h3 class="text-lg font-semibold">Directions</h3>
            <button 
              @click="mapInfo.showRouteDetails = !mapInfo.showRouteDetails" 
              class="text-sm text-blue-600 hover:underline flex items-center"
            >
              {{ mapInfo.showRouteDetails ? 'Hide' : 'Show' }}
              <span class="ml-1">{{ mapInfo.showRouteDetails ? '▲' : '▼' }}</span>
            </button>
          </div>
          
          <div v-if="mapInfo.showRouteDetails" class="mt-2 border rounded-lg overflow-hidden">
            <div class="bg-gray-50 border-b px-4 py-2 text-sm text-gray-500">
              Total {{ mapInfo.routeInfo.steps.length }} steps • {{ formattedDistance }} • {{ formattedDuration }}
            </div>
            
            <div class="max-h-[400px] overflow-y-auto">
              <div 
                v-for="(step, index) in mapInfo.routeInfo.steps" 
                :key="index"
                class="p-3 border-b last:border-b-0 hover:bg-gray-50"
              >
                <div class="flex space-x-3">
                  <div class="flex-shrink-0 h-8 w-8 bg-blue-100 rounded-full flex items-center justify-center text-blue-600 font-semibold text-sm">
                    {{ getManeuverIcon(step.maneuver) }}
                  </div>
                  <div>
                    <div v-html="step.instructions"></div>
                    <div class="text-sm text-gray-500 mt-1">
                      {{ step.distance }} • {{ step.duration }}
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
        
        <!-- Yer detayları -->
        <div v-if="mapInfo.placeDetails" class="mb-4 p-4 border rounded-lg bg-gray-50">
          <h3 class="text-lg font-semibold flex items-center">
            <span>{{ mapInfo.placeDetails.name }}</span>
            <span v-if="mapInfo.placeDetails.rating" class="ml-2 text-amber-500">⭐ {{ mapInfo.placeDetails.rating }}</span>
          </h3>
          <p class="text-gray-600 mb-1">{{ mapInfo.placeDetails.formatted_address }}</p>
          
          <div v-if="mapInfo.placeDetails.photos && mapInfo.placeDetails.photos.length" class="my-2 overflow-x-auto whitespace-nowrap">
            <img 
              v-for="(photo, index) in mapInfo.placeDetails.photos.slice(0, 3)" 
              :key="index" 
              :src="photo.getUrl({ maxWidth: 200, maxHeight: 150 })" 
              alt="Place photo" 
              class="inline-block h-32 w-auto rounded-lg mr-2"
            />
          </div>
          
          <div class="flex flex-wrap gap-2 mt-2">
            <a 
              v-if="mapInfo.placeDetails.website" 
              :href="mapInfo.placeDetails.website" 
              target="_blank"
              class="text-sm px-2 py-1 bg-blue-100 text-blue-700 rounded hover:bg-blue-200"
            >
              🌐 Website
            </a>
            <span 
              v-if="mapInfo.placeDetails.formatted_phone_number"
              class="text-sm px-2 py-1 bg-green-100 text-green-700 rounded"
            >
              📞 {{ mapInfo.placeDetails.formatted_phone_number }}
            </span>
          </div>
        </div>
        
        <h3 class="text-lg font-semibold mb-2">Markers:</h3>
        <ul class="space-y-2">
          <li v-for="(point, index) in markers" :key="point.id" 
              class="p-3 border border-gray-200 rounded-lg hover:bg-gray-50">
            <div class="flex justify-between items-center">
              <div>
                <p class="font-medium">Point {{ index + 1 }}</p>
                <p class="text-sm text-gray-600">
                  Lat: {{ point.position.lat.toFixed(6) }}, 
                  Lng: {{ point.position.lng.toFixed(6) }}
                </p>
              </div>
              <button 
                @click="deleteMarker(point.id, googleInstance)"
                class="px-2 py-1 bg-red-500 text-white text-sm rounded hover:bg-red-600"
              >
                Remove
              </button>
            </div>
          </li>
        </ul>
        
        <!-- Yakındaki yerler -->
        <div v-if="mapInfo.nearbyPlaces.length > 0" class="mt-6">
          <div class="flex justify-between items-center mb-2">
            <h3 class="text-lg font-semibold">Nearby Places:</h3>
            
            <!-- Yer tipi seçimi -->
            <div class="flex items-center space-x-2">
              <label for="place-type" class="text-sm text-gray-600">Type:</label>
              <select 
                id="place-type" 
                v-model="mapInfo.nearbyType"
                @change="findNearbyPlaces(markers[markers.length-1].marker.getPosition(), googleInstance)"
                class="text-sm border rounded p-1"
              >
                <option v-for="type in placeTypes" :key="type.value" :value="type.value">
                  {{ type.icon }} {{ type.label }}
                </option>
              </select>
            </div>
          </div>
          
          <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-2">
            <div 
              v-for="place in mapInfo.nearbyPlaces.slice(0, 6)" 
              :key="place.place_id"
              class="p-2 border rounded-lg hover:bg-blue-50 cursor-pointer"
              @click="getPlaceDetails(place.place_id, googleInstance)"
            >
              <div class="flex items-start">
                <div v-if="place.photos && place.photos.length" class="mr-2 flex-shrink-0">
                  <img :src="place.photos[0].getUrl({ maxWidth: 80, maxHeight: 80 })" alt="Place photo" class="w-16 h-16 object-cover rounded" />
                </div>
                <div>
                  <h4 class="font-semibold">{{ place.name }}</h4>
                  <p class="text-xs text-gray-600">{{ place.vicinity }}</p>
                  <div class="flex items-center mt-1 text-sm">
                    <span v-if="place.rating" class="text-amber-500 mr-1">⭐ {{ place.rating.toFixed(1) }}</span>
                    <span v-if="place.user_ratings_total" class="text-gray-500">({{ place.user_ratings_total }})</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
        
        <!-- Temizle ve Sıfırla düğmeleri -->
        <div class="flex space-x-4 mt-6">
          <button 
            @click="clearAllMarkers(googleInstance)"
            class="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700 transition"
          >
            Clear All Markers
          </button>
          
          <button 
            @click="resetMap()"
            class="px-4 py-2 bg-gray-600 text-white rounded hover:bg-gray-700 transition"
          >
            Reset Map
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* Bileşene özel stiller */
.map-control-button {
  @apply px-3 py-1 rounded-md text-sm font-medium transition-colors;
}
</style> 