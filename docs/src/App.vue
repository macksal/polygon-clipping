<template>
  <div id="app">
    <div id="map">
      <div class="control leaflet-bar">
        <h4>Input Data</h4>
        <select @change="setInput">
          <option>Asia</option>
          <option>Almost Parallel Segments</option>
          <option>Saw & Cheese</option>
        </select>
        <br /><br />
        <div v-for="operation in operations" :key="operation">
          <input
            v-model="selectedOperation"
            type="radio"
            name="some"
            :value="operation"
            @change="setOperation"
          />
          {{ operation }}
        </div>

        <h4>Performance</h4>
        <p>
          polygon-clipping {{ performance }} m/s<br />
          martinez {{ martinezPerf }} m/s<br />
          jsts {{ jstsPerf }} m/s<br />
        </p>
      </div>
    </div>
  </div>
</template>

<script>
import pc from "../.."

import {
  union as martinezUnion,
  intersection as martinezIntersection,
  diff as martinezDifference,
  xor as martinezXor,
} from "martinez-polygon-clipping"

// turf v5 runs off of jsts under the hood
import jstsUnion from "@turf/union"
import jstsIntersection from "@turf/intersect"
import jstsDifference from "@turf/difference"
var jstsXor = null

var operation = pc.intersection
var martinezOp = martinezUnion
var jstsOp = jstsUnion
var inData = null
var inLayer = null
var outLayer = null
var map = null

import asia from "../geojson/asia-with-poly.json"
import parallel from "../geojson/parallel.json"
import cheese from "../geojson/cheese.json"

export default {
  name: "App",
  data() {
    return {
      operations: ["Intersection", "Union", "Difference", "XOR"],
      selectedOperation: "Intersection",
      performance: "",
      martinezPerf: "",
      jstsPerf: "",
    }
  },
  mounted() {
    inData = asia
    map = window.map = L.map("map", {
      minZoom: 1,
      maxZoom: 20,
      center: [0, 0],
      zoom: 2,
      crs: L.CRS.Simple,
    })

    inLayer = L.geoJson(asia).addTo(map)

    map.fitBounds(inLayer.getBounds(), {
      padding: [20, 20],
    })
    outLayer = L.geoJson(
      {
        type: "FeatureCollection",
        features: [],
      },
      {
        color: "red",
      },
    ).addTo(map)

    this.runOperation()
  },
  methods: {
    setInput(e) {
      inLayer.clearLayers()
      outLayer.clearLayers()
      if (e.target.value === "Asia") inData = asia
      if (e.target.value === "Almost Parallel Segments") inData = parallel
      if (e.target.value === "Saw & Cheese") inData = cheese
      inLayer.addData(inData)
      map.fitBounds(inLayer.getBounds(), {
        padding: [20, 20],
      })
      this.runOperation()
    },
    setOperation(e) {
      this.selectedOperation = e.target.value
      outLayer.clearLayers()

      if (this.selectedOperation === "Union") {
        operation = pc.union
        martinezOp = martinezUnion
        jstsOp = jstsUnion
      }
      if (this.selectedOperation === "Intersection") {
        operation = pc.intersection
        martinezOp = martinezIntersection
        jstsOp = jstsIntersection
      }
      if (this.selectedOperation === "XOR") {
        operation = pc.xor
        martinezOp = martinezXor
        jstsOp = jstsXor
      }
      if (this.selectedOperation === "Difference") {
        operation = pc.difference
        martinezOp = martinezDifference
        jstsOp = jstsDifference
      }

      this.runOperation()
    },
    runOperation() {
      var t0 = performance.now()
      var outData = operation(
        inData.features[0].geometry.coordinates,
        inData.features[1].geometry.coordinates,
      )
      this.performance = (performance.now() - t0).toFixed(2)

      outLayer
        .addData({
          type: "MultiPolygon",
          coordinates: outData,
        })
        .addTo(map)

      var m0 = performance.now()
      martinezOp(
        inData.features[0].geometry.coordinates,
        inData.features[1].geometry.coordinates,
      )
      this.martinezPerf = (performance.now() - m0).toFixed(2)

      if (jstsOp !== null) {
        var j0 = performance.now()
        jstsOp(inData.features[0], inData.features[1])
        this.jstsPerf = (performance.now() - j0).toFixed(2)
      } else {
        this.jstsPerf = "N/A"
      }
    },
  },
}
</script>

<style>
html,
body,
#app,
#map {
  width: 100%;
  height: 100%;
  margin: 0px;
}

.control {
  position: absolute;
  top: 10px;
  right: 10px;
  background: white;
  padding: 10px;
  z-index: 10000;
}

h4 {
  margin-bottom: 5px;
}
p {
  margin-top: 5px;
}
</style>
