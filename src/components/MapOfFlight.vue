<template>
  <div class="wrapper">
    <div ref="mapContainer" id="maps"></div>
    <button @click="changeMovement" class="button">{{ buttontText }}</button>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import 'ol/ol.css'
import Map from 'ol/Map'
import View from 'ol/View'
import { Tile as TileLayer } from 'ol/layer'
import { OSM, XYZ } from 'ol/source'
import { Feature } from 'ol'
import { Point } from 'ol/geom'
import { Vector as VectorLayer } from 'ol/layer'
import { Vector as VectorSource } from 'ol/source'
import { Style, Icon } from 'ol/style'

const mapContainer = ref(null)
const map = ref(null)
const marker = ref(null)
const latitude = ref(0)
const longitude = ref(0)
const isMovementRunning = ref(false)
const timeoutId = ref(null)
import data from '../service/data.json'
const buttontText = computed(() => (isMovementRunning.value ? 'Стоп' : 'Почати'))

const updateMarkerCoordinates = (coordinatesData) => {
  let index = 0
  const delay = 200

  const updateNextMarker = () => {
    if (index < coordinatesData.length) {
      const { latitude, longitude } = coordinatesData[index]
      marker.value.getGeometry().setCoordinates([longitude, latitude])
      map.value.getView().setCenter([longitude, latitude])
      index++
      timeoutId.value = setTimeout(updateNextMarker, delay)
    }
  }

  updateNextMarker()
}

const startMovement = () => {
  const coordinates = data.map(({ speed, direction, timestamp }) =>
    calculateNewCoordinates(speed, direction, timestamp)
  )
  updateMarkerCoordinates(coordinates)
  isMovementRunning.value = true
}

const stopMovement = () => {
  latitude.value = 0
  longitude.value = 0

  marker.value.getGeometry().setCoordinates([longitude.value, latitude.value])
  map.value.getView().setCenter([longitude.value, latitude.value])

  clearTimeout(timeoutId.value)
  timeoutId.value = null
  isMovementRunning.value = false
}

const changeMovement = () => {
  if (isMovementRunning.value) {
    stopMovement()
  } else {
    startMovement()
  }
}

const calculateNewCoordinates = (speed, direction, timestamp) => {
  const directionInRadians = (parseFloat(direction) * Math.PI) / 180
  const speedPerSecond = parseFloat(speed) / (60 * 60 * 1000)

  const distanceOfTravel = speedPerSecond * timestamp

  const distanceLatitude = distanceOfTravel * Math.cos(directionInRadians)
  const distanceLongitude = distanceOfTravel * Math.sin(directionInRadians)

  const newLatitude = latitude.value + distanceLatitude
  const newLongitude = longitude.value + distanceLongitude

  latitude.value = newLatitude
  longitude.value = newLongitude

  return { latitude: newLatitude, longitude: newLongitude }
}

onMounted(() => {
  map.value = new Map({
    target: mapContainer.value,
    layers: [
      new TileLayer({
        source: new OSM()
      }),
      new TileLayer({
        source: new XYZ({
          url: 'http://server.arcgisonline.com/ArcGIS/rest/services/World_Physical_Map/MapServer/tile/{z}/{y}/{x}'
        })
      })
    ],
    view: new View({
      center: [0, 0],
      zoom: 2
    })
  })

  const customIcon = new Icon({
    src: '/img/plane.png',
    anchor: [0.5, 1],
    scale: 1
  })

  marker.value = new Feature({
    geometry: new Point([0, 0])
  })

  const markerStyle = new Style({
    image: customIcon
  })

  marker.value.setStyle(markerStyle)

  const vectorLayer = new VectorLayer({
    source: new VectorSource({
      features: [marker.value]
    })
  })

  map.value.addLayer(vectorLayer)
})
</script>

<style scoped>
#maps {
  width: 700px;
  height: 500px;
  border: 5px solid black;
}
.wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  position: relative;
}
.button {
  background: yellow;
  text-transform: uppercase;
  width: 100px;
  border: none;
  padding: 5px;
  position: absolute;
  bottom: 150px;
}
</style>
