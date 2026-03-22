<script setup lang="ts">
import { ref } from 'vue'

const US_STATES = [
  'AL','AK','AZ','AR','CA','CO','CT','DE','FL','GA',
  'HI','ID','IL','IN','IA','KS','KY','LA','ME','MD',
  'MA','MI','MN','MS','MO','MT','NE','NV','NH','NJ',
  'NM','NY','NC','ND','OH','OK','OR','PA','RI','SC',
  'SD','TN','TX','UT','VT','VA','WA','WV','WI','WY',
  'DC',
]

interface LookupResult {
  make: string
  model: string
  year: string
  imageUrl: string | null
}

const plate = ref('')
const state = ref('MA')
const loading = ref(false)
const error = ref<string | null>(null)
const result = ref<LookupResult | null>(null)

const API_KEY = import.meta.env.VITE_CARSXE_API_KEY as string

const examples = [
  { plate: '1WKD71', state: 'MA' },
  { plate: 'JF248A', state: 'MA' },
  { plate: '6BF450', state: 'MA' },
]

async function lookup() {
  if (!plate.value.trim()) return

  loading.value = true
  error.value = null
  result.value = null

  try {
    const decodeUrl = new URL('/api/carsxe/v2/platedecoder', window.location.origin)
    decodeUrl.searchParams.set('key', API_KEY)
    decodeUrl.searchParams.set('plate', plate.value.trim())
    decodeUrl.searchParams.set('state', state.value)

    const decodeRes = await fetch(decodeUrl.toString())
    if (!decodeRes.ok) throw new Error(`Plate lookup failed: ${decodeRes.status}`)
    const decodeData = await decodeRes.json()

    if (decodeData.error) throw new Error(decodeData.error)

    const make: string = decodeData.make ?? decodeData.attributes?.make ?? ''
    const model: string = decodeData.model ?? decodeData.attributes?.model ?? ''
    const year: string = String(decodeData.year ?? decodeData.attributes?.year ?? '')

    if (!make || !model) throw new Error('Could not determine make/model for this plate.')

    const imageUrl = await fetchImage(make, model, year)

    result.value = { make, model, year, imageUrl }
  } catch (e) {
    error.value = e instanceof Error ? e.message : 'An unknown error occurred.'
  } finally {
    loading.value = false
  }
}

async function fetchImage(make: string, model: string, year: string): Promise<string | null> {
  try {
    const imageUrl = new URL('/api/carsxe/images', window.location.origin)
    imageUrl.searchParams.set('key', API_KEY)
    imageUrl.searchParams.set('make', make)
    imageUrl.searchParams.set('model', model)
    if (year) imageUrl.searchParams.set('year', year)
    imageUrl.searchParams.set('format', 'json')

    const imageRes = await fetch(imageUrl.toString())
    if (!imageRes.ok) return null
    const imageData = await imageRes.json()

    return imageData.images?.[0]?.link ?? null
  } catch {
    return null
  }
}
</script>

<template>
  <div class="layout">
    <!-- Left pane: input -->
    <div class="pane pane-left">
      <div class="form-wrapper">
        <h1>License Plate Lookup</h1>
        <form @submit.prevent="lookup">
          <div class="inputs">
            <input
              v-model="plate"
              class="plate-input"
              type="text"
              placeholder="e.g. 1WKD71"
              maxlength="10"
              autocomplete="off"
              spellcheck="false"
            />
            <select v-model="state" class="state-select">
              <option v-for="s in US_STATES" :key="s" :value="s">{{ s }}</option>
            </select>
          </div>
          <button class="btn" type="submit" :disabled="loading || !plate.trim()">
            {{ loading ? 'Looking up…' : 'Look up' }}
          </button>
        </form>
        <p v-if="error" class="error">{{ error }}</p>

        <table class="examples">
          <tbody>
            <tr v-for="ex in examples" :key="ex.plate" @click="plate = ex.plate">
              <td class="ex-plate">{{ ex.plate }}</td>
              <td class="ex-state">{{ ex.state }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Right pane: result -->
    <div class="pane pane-right">
      <div v-if="result" class="result">
        <img
          v-if="result.imageUrl"
          :src="result.imageUrl"
          :alt="`${result.year} ${result.make} ${result.model}`"
          class="car-image"
        />
        <div v-else class="no-image">No image available</div>
        <div class="vehicle-info">
          <p class="vehicle-year">{{ result.year }}</p>
          <p class="vehicle-name">{{ result.make }} {{ result.model }}</p>
        </div>
      </div>
      <div v-else class="empty-pane">
        <p>Results will appear here</p>
      </div>
    </div>
  </div>
</template>

<style scoped>
.layout {
  display: flex;
  height: 100vh;
  font-family: system-ui, sans-serif;
}

.pane {
  flex: 1;
  overflow: hidden;
}

.pane-left {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
  border-right: 1px solid #e5e7eb;
}

.pane-right {
  display: flex;
  align-items: center;
  justify-content: center;
  background: #f9fafb;
}

.form-wrapper {
  width: 100%;
  max-width: 360px;
}

h1 {
  font-size: 1.5rem;
  font-weight: 700;
  margin: 0 0 24px;
}

.inputs {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
}

.plate-input {
  flex: 1;
  padding: 10px 14px;
  font-size: 1rem;
  font-weight: 600;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  border: 2px solid #d1d5db;
  border-radius: 8px;
  outline: none;
  transition: border-color 0.15s;
}

.plate-input:focus {
  border-color: #3b82f6;
}

.state-select {
  padding: 10px 12px;
  font-size: 1rem;
  border: 2px solid #d1d5db;
  border-radius: 8px;
  background: #fff;
  cursor: pointer;
  outline: none;
}

.state-select:focus {
  border-color: #3b82f6;
}

.btn {
  width: 100%;
  padding: 11px;
  font-size: 1rem;
  font-weight: 600;
  color: #fff;
  background: #3b82f6;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s;
}

.btn:hover:not(:disabled) {
  background: #2563eb;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.error {
  color: #dc2626;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  padding: 12px 16px;
  margin-top: 16px;
}

/* Right pane result */
.result {
  width: 100%;
  max-width: 480px;
  padding: 32px;
}

.car-image {
  width: 100%;
  border-radius: 12px;
  object-fit: cover;
  display: block;
  box-shadow: 0 4px 16px rgba(0,0,0,0.1);
}

.no-image {
  height: 220px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: #e5e7eb;
  border-radius: 12px;
  color: #9ca3af;
  font-size: 0.9rem;
}

.vehicle-info {
  margin-top: 20px;
}

.vehicle-year {
  font-size: 0.9rem;
  color: #6b7280;
  margin: 0 0 4px;
}

.vehicle-name {
  font-size: 1.75rem;
  font-weight: 700;
  margin: 0;
}

.empty-pane {
  color: #9ca3af;
  font-size: 0.95rem;
}

.examples {
  margin-top: 24px;
  width: 100%;
  border-collapse: collapse;
  font-size: 0.8rem;
  color: #9ca3af;
}

.examples tr {
  cursor: pointer;
  transition: color 0.1s;
}

.examples tr:hover {
  color: #3b82f6;
}

.ex-plate {
  padding: 4px 0;
  font-family: monospace;
  letter-spacing: 0.05em;
}

.ex-state {
  padding: 4px 0;
  text-align: right;
}
</style>
