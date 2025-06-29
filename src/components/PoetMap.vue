<template>
  <div class="app-container">
    <header class="header">
      <p class="range-label">Seçili Aralık: {{ rangeLabel }}</p>
    </header>

    <div id="map" class="map"></div>

    <div ref="sliderEl" class="slider"></div>

    <div class="bottom-panel">
      <p class="panel-label">&nbsp;</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue';
import 'leaflet/dist/leaflet.css';
import noUiSlider from 'nouislider';
import 'nouislider/dist/nouislider.css';
import poets from '../data/poets.json'; // JSON dosyanız burada olacak

const sliderEl = ref(null);

const YEAR_MIN = 1000;
const YEAR_MAX = 1950;
const STEP = 50;

const range = ref([YEAR_MIN, YEAR_MIN]);
const rangeLabel = computed(() => `${range.value[0]} – ${range.value[1]}`);

let map = null;
let markers = [];

function parsePeriod(period) {
  const match = period?.match(/(\d{2})\.\s?yy/);
  return match ? (parseInt(match[1]) - 1) * 100 : null;
}

function getBirthYear(poet) {
  return poet.birth_year ?? parsePeriod(poet.period);
}

function filterPoets() {
  return poets.filter((p) => {
    const birth = getBirthYear(p);
    return birth && birth >= range.value[0] && birth < range.value[1];
  });
}

function jitter([lat, lng], idx) {
  if (idx === 0) return [lat, lng]; // İlk kişi orijinal konumda kalsın

  const R = 0.001; // Derece cinsinden (0.15 = ~15 km)
  const angle = idx * 10 * (Math.PI / 90); // 10° aralıklarla dağıt
  const dLat = R * Math.cos(angle);
  const dLng = R * Math.sin(angle);
  return [lat + dLat, lng + dLng];
}

function updateMarkers(L) {
  markers.forEach((m) => m.remove());
  markers = [];

  const locIndex = {}; // aynı lokasyondaki sayıyı takip etmek için

  filterPoets().forEach((poet) => {
    if (!poet.location) return;

    const key = poet.location.join(',');
    const idx = locIndex[key] ?? 0;
    const coords = jitter(poet.location, idx);
    locIndex[key] = idx + 1;

    const marker = L.marker(coords).addTo(map);
    const content = `
      <strong>${poet.name}</strong><br/>
      ${poet.birth_year ?? poet.period} – ${poet.death_year ?? '?'}<br/>
      <em>${poet.style}</em><br/>
      <small>${poet.notes}</small>
    `;
    marker.bindPopup(content);
    markers.push(marker);
  });
}

onMounted(async () => {
  const L = await import('leaflet');
  await nextTick();

  map = L.map('map').setView([39, 50], 5);

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '&copy; OpenStreetMap contributors',
    maxZoom: 18,
    detectRetina: true,
  }).addTo(map);

  if (sliderEl.value) {
    noUiSlider.create(sliderEl.value, {
      start: range.value,
      connect: true,
      step: STEP,
      range: { min: YEAR_MIN, max: YEAR_MAX },
      tooltips: true,
      format: {
        to: (v) => Math.round(v),
        from: (v) => Number(v),
      },
    });

    sliderEl.value.noUiSlider.on('update', (vals) => {
      range.value = vals.map(Number);
      updateMarkers(L);
    });
  }

  updateMarkers(L);
});
</script>

<style scoped>
@import 'nouislider/dist/nouislider.css';

html,
body,
#app {
  height: 100vh;
  width: 100vw;
  margin: 0;
  padding: 0;
  overflow: hidden;
}

.app-container {
  height: 100vh;
  width: 100vw;
  display: flex;
  flex-direction: column;
}

.header {
  background-color: #f3f4f6;
  padding: 1rem;
  box-shadow: 0 1px 2px rgb(0 0 0 / 0.1);
  height: 30px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.range-label {
  font-size: 0.875rem;
  color: #4b5563;
}

.map {
  flex: 1;
  width: 100%;
}

.slider {
  position: fixed;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%);
  width: 80%;
  max-width: 600px;
  z-index: 1000;
}

/* Tooltip'leri her zaman tutamacın ALTINA sabitle */
.slider .noUi-horizontal .noUi-handle .noUi-tooltip {
  top: 100% !important;
  transform: translate(-50%, 10px) !important;
  margin: 0 !important;
  position: absolute !important;
  z-index: 10;
  white-space: nowrap;
  min-width: 50px;
  text-align: center;
}

.bottom-panel {
  position: fixed;
  bottom: 0;
  left: 0;
  width: 100%;
  background-color: #f3f4f6;
  padding: 10px;
  box-shadow: 0 -1px 2px rgb(0 0 0 / 0.1);
  text-align: center;
  z-index: 999;
}
</style>
