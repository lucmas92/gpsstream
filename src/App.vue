<script setup>
import { ref } from 'vue'

const location = ref(null)
const error = ref(null)
const loading = ref(false)
const history = ref([])
const isTracking = ref(false)
const watchId = ref(null)
const message = ref('')

// Funzione per calcolare la distanza tra due punti GPS (formula Haversine)
function getDistance(lat1, lon1, lat2, lon2) {
  const R = 6371 // Raggio della Terra in km
  const dLat = (lat2 - lat1) * Math.PI / 180
  const dLon = (lon2 - lon1) * Math.PI / 180
  const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
            Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
            Math.sin(dLon / 2) * Math.sin(dLon / 2)
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a))
  return R * c
}

// Funzione per elaborare la posizione ricevuta
async function processLocation(pos) {
  const timestamp = new Date(pos.timestamp)
  const data = {
    userId: "user_001",
    latitude: pos.coords.latitude,
    longitude: pos.coords.longitude,
    accuracy: pos.coords.accuracy,
    altitude: pos.coords.altitude,
    timestamp: timestamp.toISOString()
  }

  location.value = data
  loading.value = false

  // Controlla se le coordinate sono diverse dall'ultima posizione
  const isDifferent = history.value.length === 0 ||
    Math.abs(data.latitude - history.value[history.value.length - 1].latitude) >= 0.0001 ||
    Math.abs(data.longitude - history.value[history.value.length - 1].longitude) >= 0.0001

  if (!isDifferent) {
    message.value = "Coordinate invariate, nessuna nuova posizione aggiunta."
    return
  }

  message.value = ''

  // Calcola velocità se c'è una posizione precedente
  let speed = 0
  if (history.value.length > 0) {
    const prev = history.value[history.value.length - 1]
    const distance = getDistance(prev.latitude, prev.longitude, data.latitude, data.longitude)
    const deltaTime = (timestamp - new Date(prev.timestamp)) / 1000 / 3600 // ore
    speed = deltaTime > 0 ? distance / deltaTime : 0 // km/h
  }

  // Aggiungi alla history
  history.value.push({
    latitude: data.latitude,
    longitude: data.longitude,
    timestamp: data.timestamp,
    speed: speed.toFixed(2)
  })

  // Mantieni solo le ultime 20 posizioni
  if (history.value.length > 20) {
    history.value.shift()
  }

  // 👉 invio backend
  try {
    await fetch("https://gpsstream.shop/service/trigger/gps", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-API-KEY":"chiave"
      },
      body: JSON.stringify(data)
    })
  } catch (e) {
    console.error("Errore invio:", e)
  }
}

async function getLocation() {
  if (!navigator.geolocation) {
    error.value = "Geolocalizzazione non supportata"
    return
  }

  loading.value = true
  error.value = null

  navigator.geolocation.getCurrentPosition(
      processLocation,
      (err) => {
        error.value = err.message
        loading.value = false
      }
  )
}

function startTracking() {
  if (isTracking.value) return
  isTracking.value = true
  getLocation() // Prima posizione immediata
  watchId.value = navigator.geolocation.watchPosition(processLocation, (err) => {
    error.value = err.message
    loading.value = false
  }, {
    enableHighAccuracy: true,
    maximumAge: 10000,
    timeout: 5000
  })
}

function stopTracking() {
  if (!isTracking.value) return
  isTracking.value = false
  if (watchId.value !== null) {
    navigator.geolocation.clearWatch(watchId.value)
    watchId.value = null
  }
}
</script>

<template>
  <div style="padding:20px; font-family:sans-serif;">
    <h1>📍 GPS Tracker</h1>

    <button @click="startTracking" :disabled="isTracking || loading">
      {{ isTracking ? "Tracciamento attivo..." : "Inizia Tracciamento" }}
    </button>
    <button @click="stopTracking" :disabled="!isTracking || loading">
      Ferma Tracciamento
    </button>

    <p v-if="error" style="color:red;">
      Errore: {{ error }}
    </p>
    <p v-if="message" style="color:rgb(0 245 227);">
      {{ message }}
    </p>

    <div v-if="history.length > 0">
      <h2>Storico Posizioni</h2>
      <table style="width:100%; border-collapse:collapse;">
        <thead>
          <tr>
            <th style="border:1px solid black; padding:8px;">Timestamp</th>
            <th style="border:1px solid black; padding:8px;">Latitudine</th>
            <th style="border:1px solid black; padding:8px;">Longitudine</th>
            <th style="border:1px solid black; padding:8px;">Velocità (km/h)</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(pos, index) in history.slice().reverse()" :key="index">
            <td style="border:1px solid black; padding:8px;">{{ pos.timestamp }}</td>
            <td style="border:1px solid black; padding:8px;">{{ pos.latitude.toFixed(6) }}</td>
            <td style="border:1px solid black; padding:8px;">{{ pos.longitude.toFixed(6) }}</td>
            <td style="border:1px solid black; padding:8px;">{{ pos.speed }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>