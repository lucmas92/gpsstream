<script setup>
import { ref, onMounted } from 'vue'

const location = ref(null)
const error = ref(null)
const loading = ref(false)
const history = ref([])
const isTracking = ref(false)
const watchId = ref(null)
const message = ref('')
const lastSendTime = ref(0)

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
  console.log('processLocation', pos)
  const timestamp = new Date(pos.timestamp)
  const data = {
    userId: "user_001",
    latitude: pos.coords.latitude,
    longitude: pos.coords.longitude,
    accuracy: pos.coords.accuracy,
    altitude: pos.coords.altitude,
    timestamp: timestamp.toISOString()
  }

  // Verifica se la posizione è cambiata significativamente
  if (location.value && location.value.latitude === data.latitude && location.value.longitude === data.longitude) {
    message.value = 'Posizione non cambiata'
    return
  } else {
    message.value = ''
  }

  location.value = data
  loading.value = false

  // Controllo frequenza invio: al massimo 1 ogni 2 secondi
  if (Date.now() - lastSendTime.value < 2000) {
    message.value = 'Invio troppo frequente, attendi'
    return
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
    lastSendTime.value = Date.now()
    // Dopo l'invio, aggiorna lo storico
    await fetchHistory()
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

// Funzione per recuperare lo storico dal server
async function fetchHistory() {
  try {
    const response = await fetch("https://gpsstream.shop/service/get-gps", {
      method: "GET",
      headers: {
        "X-API-KEY":"chiave"
      },
    })

    if (!response.ok) throw new Error('Errore nel recupero dello storico')
    const data = await response.json()
    // Ordina per timestamp decrescente (più recenti prima)
    const sortedData = data.sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp))

    // Calcola velocità per ogni posizione
    for (let i = 0; i < sortedData.length; i++) {
      if (i > 0) {
        const prev = sortedData[i - 1]
        const curr = sortedData[i]
        const distance = getDistance(prev.latitude, prev.longitude, curr.latitude, curr.longitude)
        const deltaTime = (new Date(prev.timestamp) - new Date(curr.timestamp)) / 1000 / 3600 // ore
        sortedData[i].speed = deltaTime > 0 ? (distance / deltaTime).toFixed(2) : 0
      } else {
        sortedData[i].speed = 0 // Prima posizione, velocità 0
      }
    }

    history.value = sortedData
  } catch (e) {
    console.error("Errore recupero storico:", e)
    error.value = "Errore nel recupero dello storico"
  }
}

// Funzione per formattare il timestamp in data e ora locali
function formatTimestamp(ts) {
  const date = new Date(ts)
  return date.toLocaleString()
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

// Carica lo storico al montaggio del componente
onMounted(() => {
  fetchHistory()
})
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
    <p v-if="message" style="color:blue;">
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
            <th style="border:1px solid black; padding:8px;">Precisione</th>
            <th style="border:1px solid black; padding:8px;">Velocità (km/h)</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="pos in history" :key="pos.id">
            <td style="border:1px solid black; padding:8px;">{{ formatTimestamp(pos.timestamp) }}</td>
            <td style="border:1px solid black; padding:8px;">{{ pos.latitude.toFixed(6) }}</td>
            <td style="border:1px solid black; padding:8px;">{{ pos.longitude.toFixed(6) }}</td>
            <td style="border:1px solid black; padding:8px;">{{ pos.accuracy.toFixed(2) }} m</td>
            <td style="border:1px solid black; padding:8px;">{{ pos.speed }} km/h</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>