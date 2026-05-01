<script setup>
import { ref } from 'vue'

const location = ref(null)
const error = ref(null)
const loading = ref(false)

async function getLocation() {
  if (!navigator.geolocation) {
    error.value = "Geolocalizzazione non supportata"
    return
  }

  loading.value = true
  error.value = null

  navigator.geolocation.getCurrentPosition(
      async (pos) => {
        const data = {
          userId: "user_001",
          latitude: pos.coords.latitude,
          longitude: pos.coords.longitude,
          accuracy: pos.coords.accuracy,
          altitude: pos.coords.altitude,
          timestamp: new Date(pos.timestamp).toISOString()
        }

        location.value = data
        loading.value = false

        // 👉 invio backend
        try {
          await fetch("/api/location", {
            method: "POST",
            headers: {
              "Content-Type": "application/json"
            },
            body: JSON.stringify(data)
          })
        } catch (e) {
          console.error("Errore invio:", e)
        }
      },
      (err) => {
        error.value = err.message
        loading.value = false
      }
  )
}
</script>

<template>
  <div style="padding:20px; font-family:sans-serif;">
    <h1>📍 GPS Tracker</h1>

    <button @click="getLocation" :disabled="loading">
      {{ loading ? "Recupero..." : "Ottieni posizione" }}
    </button>

    <p v-if="error" style="color:red;">
      Errore: {{ error }}
    </p>

    <pre v-if="location">
{{ JSON.stringify(location, null, 2) }}
    </pre>
  </div>
</template>