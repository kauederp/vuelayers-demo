<template>
  <div>
    <vl-map 
      ref="map"
      :load-tiles-while-animating="true"
      :load-tiles-while-interacting="true"
      data-projection="EPSG:3857"
      :style="style"
    >
      <vl-view
        ref="mapView"
        :zoom.sync="zoom"
        :center.sync="center"
        :rotation.sync="rotation"
      />

      <vl-layer-tile
        v-if="baseLayer !== undefined"
        :id="baseLayer.id"
        :z-index="0"
      >
        <component :is="'vl-source-' + baseLayer.name" v-bind="baseLayer" />
      </vl-layer-tile>

      <vl-layer-vector id="drawLayer" :z-index="1">
        <vl-source-vector ident="drawTarget" :features.sync="features" />
      </vl-layer-vector>
      <vl-interaction-draw
        ref="interactionDraw"
        v-if="drawType != undefined"
        source="drawTarget"
        :type="drawType"
      />

      <vl-layer-vector v-if="drawType == undefined" id="editLayer" :z-index="5">
        <vl-source-vector
          ident="editTarget"
          :features.sync="featuresSelected"
        ></vl-source-vector>
      </vl-layer-vector>
      <vl-interaction-select
        v-if="drawType == undefined"
        :features.sync="featuresSelected"
        source="editTarget"
      />
      <div v-for="layer in layers" :key="layer.id" :id="layer.id">
        <vl-layer-vector
          :visible="layer.properties.visible"
          :zIndex="layer.zIndex"
        >
          <vl-source-vector :ref="layer.properties.name">
            <vl-feature :properties="layer.properties">
              <component
                :is="'vl-geom-'+this.drawType"
                v-if="layer.geometry != null"
                :coordinates="layer.geometry.coordinates"
              />
            </vl-feature>
          </vl-source-vector>
          <vl-style-box v-if="layer.style">
            <vl-style-stroke
              :color="layer.style.strokeColor"
              :width="layer.style.strokeWidth"
            />
            <vl-style-fill :color="layer.style.fillColor" />
          </vl-style-box>
        </vl-layer-vector>
      </div>

      <!-- <vl-layer-vector v-bind="layers.limit1"> 
        <vl-source-vector ref="limit1Source"
          :url="layers.limit1.source.url" 
          :features.sync="layers.limit1.source.features">
      </vl-layer-vector> -->
    </vl-map>
  </div>
</template>

<script>

export default {
  name: "Map",
  props: {
    baseLayer: Object,
    drawType: String,
    layers: Object,
  },
  created: function () {
    window.addEventListener(
      "orientationchange",
      function () {
        window.location.reload();
      },
      false
    );
  },
  mounted() {
    window.addEventListener("load", () => {
      // do after mount (if there is a v-if in render, it probably wont work)
      this.delay(1000).then(() => {
        // wait 1 seconds before zoom to limit
        this.zoomToLimit();
      });
    });
  },
  
  data() {
    return {
      style: "height: " + window.innerHeight + "px",
      features: [],
      featuresSelected: [],
      draw: {
        type: undefined,
        features: [],
        selected: [],
      },
      // lastDrawnFeature: '',
      zoom: 2,
      center: [0, 0],
      rotation: 0,
      geolocPosition: undefined,
    };
  },
  methods: {
    delay: function (time) {
      return new Promise((resolve) => setTimeout(resolve, time));
    },
    deleteDraws: function () {
      //console.log('deleteDraws:',this.features)
      this.features = [];
    },
    zoomToLimit: function () {
      this.$refs.mapView.$view.fit(this.$refs.limit[0].$source.getExtent(), {
        size: this.$refs.map.$map.getSize(),
        duration: 1000,
      });
    },
  },
  watch: {
    features: {
      handler(val, oldVal) {
        let n = val.length;
        if (n > oldVal.length && n > 0) {
          this.$emit("map", ["new_feature", val]);
          console.log("Aqui será ultimo ponto automatico...");
        }
      },
    },
    featuresSelected: {
      handler(val, oldVal) {
        if (val.length != oldVal.length) {
          if (val.length > oldVal.length) {
            this.$emit("map", ["select_feature", val]);
          } else {
            this.$emit("map", ["unselect_feature", val]);
          }
        }
      },
    },
  },
};
</script>