<template>
  <div data-component="Map">
    <l-map :zoom="zoom" :center="center" :options="{ scrollWheelZoom: false, zoomControl: false }"
      attribution-control="false">
      <l-tile-layer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
      <l-marker :lat-lng="center">
        <l-popup>{{ location }}</l-popup>
      </l-marker>
    </l-map>
  </div>
</template>

<script lang="ts" setup>
import { ref, watch } from 'vue';
import {
  LMap,
  LTileLayer,
  LMarker,
  LPopup,
  useMap,
} from '@vue-leaflet/vue-leaflet';
import type { LatLngTuple } from 'leaflet';

// Props
const props = defineProps<{
  center: LatLngTuple;
  location?: string;
}>();

// Defaults
const zoom = ref(11);
const location = props.location || 'My Location';

// Reactive map reference for programmatic view changes
const map = ref<any>(null);

// Watch for changes to `center` and update the map's view
watch(
  () => props.center,
  (newCenter) => {
    if (map.value) {
      map.value.setView(newCenter, zoom.value);
    }
  }
);
</script>

<style scoped>
/* Add your styles here or import from './Map.scss' */
@import './Map.scss';
</style>
