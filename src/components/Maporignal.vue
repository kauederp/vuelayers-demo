<template>
  <v-container pa-0 fluid>
    <vl-map ref="map" :load-tiles-while-animating="true" :load-tiles-while-interacting="true" data-projection="EPSG:3857" :style="style">
      
      <vl-view ref="mapView" :zoom.sync="zoom" :center.sync="center" :rotation.sync="rotation"/>
      
      <vl-layer-tile v-if="baseLayer!==undefined" :id="baseLayer.id"  :z-index="0">
        <component :is="'vl-source-' + baseLayer.name" v-bind="baseLayer"/>
      </vl-layer-tile>
      
      <vl-layer-vector id="drawLayer" :z-index="1">
        <vl-source-vector ident="drawTarget" :features.sync="features"/>
      </vl-layer-vector>
      <vl-interaction-draw ref="interactionDraw" v-if="drawType!=undefined" source="drawTarget" :type="drawType"/>
      
      <vl-layer-vector v-if="drawType==undefined" id="editLayer" :z-index="5">
        <vl-source-vector ident="editTarget" :features.sync="featuresSelected"></vl-source-vector>
      </vl-layer-vector>
      <vl-interaction-select v-if="drawType==undefined" :features.sync="featuresSelected" source="editTarget"/>
      <!-- <vl-interaction-modify source="editTarget"></vl-interaction-modify>
      <vl-interaction-snap source="editTarget" :priority="10"></vl-interaction-snap> -->
      
      <!-- <vl-feature>
        <vl-geom-polygon :coordinates=coord></vl-geom-polygon>
      </vl-feature> -->

      

      <!-- <vl-layer-vector> NOT USED BECAUSE SYNC DO NOT SEND HEADER TO CHECK USER PERMISSION
        <vl-source-vector v-bind="layer1"/>
      </vl-layer-vector> -->
      
      <!-- <div v-for="layer in layers" :key="layer.id" :id="layer.id" :visible="layer.visible">
        {{layer}}<br><br>
        <component :is="layer.type" v-bind="layer">
          <component v-if="layer.source" :is="layer.source.type" :ref="layer.source.ref" :url="layer.source.url" :features.sync="layer.source.features"/>
          <component v-if="layer.style" :is="layer.style.type">
            <component v-if="layer.style.stroke" :is="layer.style.stroke.type" v-bind="layer.style.stroke"/>
            <component v-if="layer.style.fill" :is="layer.style.fill.type" v-bind="layer.style.fill"/>
          </component>
        </component>
      </div> -->

      <!-- <vl-layer-vector v-if="layers.limit" v-bind="layers.limit">
        <vl-source-vector :url="layers.limit.source.url" :features.sync="layers.limit.source.features"></vl-source-vector>
        <vl-style-box>
          <vl-style-stroke color="green" :width="3"></vl-style-stroke>
          <vl-style-fill color="rgba(255,255,255,0.5)"></vl-style-fill>
        </vl-style-box>
      </vl-layer-vector> -->
      <div v-for="layer in layers" :key="layer.id" :id="layer.id">
        <vl-layer-vector :visible="layer.properties.visible" :zIndex="layer.zIndex">
          <vl-source-vector :ref="layer.properties.name">
            <vl-feature :properties="layer.properties">
              <component :is="geometryTypeToCmpName(layer.geometry.type)" v-if="layer.geometry!=null" :coordinates="layer.geometry.coordinates" />
            </vl-feature>
          </vl-source-vector>
          <vl-style-box v-if="layer.style">
            <vl-style-stroke :color="layer.style.strokeColor" :width="layer.style.strokeWidth"/>
            <vl-style-fill :color="layer.style.fillColor"/>
          </vl-style-box>
        </vl-layer-vector>
      </div>

      <!-- <vl-layer-vector v-bind="layers.limit1"> 
        <vl-source-vector ref="limit1Source"
          :url="layers.limit1.source.url" 
          :features.sync="layers.limit1.source.features">
      </vl-layer-vector> -->
      
    </vl-map>
  </v-container>
</template>

<script>
  import { kebabCase } from 'lodash'
  //import {URLS} from '@/constants'
  //import { start} from 'vuelayers/dist/ol-ext'
  
  export default {
    name: 'Map',
    props: {
      baseLayer: Object,
      drawType: String,
      layers: Object
    },
    created: function() {
      window.addEventListener("orientationchange", function() {window.location.reload()}, false);
    },
    mounted () {
      window.addEventListener('load', () => { // do after mount (if there is a v-if in render, it probably wont work)
        this.delay(1000).then(() => { // wait 1 seconds before zoom to limit
          this.zoomToLimit()
        })
      })
    },
    data() {
      return {
        style: "height: "+window.innerHeight+"px",
        //coord: [ [ [ -43.915412209250718, -19.139507000074328 ], [ -43.54574723677203, -19.083769246996102 ], [ -43.54574723677203, -19.083769246996102 ], [ -43.345184326171889, -19.273200767895986 ], [ -43.514286388050444, -19.632886918987417 ], [ -43.978333906693898, -19.369689629688306 ], [ -43.915412209250718, -19.139507000074328 ] ], [ [ -43.730579723011381, -19.172940625314627 ], [ -43.880018754438936, -19.273200767895986 ], [ -43.78170360218396, -19.388238642115184 ], [ -43.644062389026999, -19.328874372644449 ], [ -43.644062389026999, -19.328874372644449 ], [ -43.730579723011381, -19.172940625314627 ] ], [ [ -43.561477661132827, -19.210081142946301 ], [ -43.585073297674015, -19.410494667589539 ], [ -43.486758145419046, -19.39936703539373 ], [ -43.471027721058256, -19.288048914615889 ], [ -43.561477661132827, -19.210081142946301 ] ] ],
        features: [],
        featuresSelected: [],
        draw: {
          type: undefined,
          features: [],
          selected: []
        },
        // lastDrawnFeature: '',
        zoom: 2,
        center: [0, 0],
        rotation: 0,
        geolocPosition: undefined
      };
    },
    methods: {
      delay: function(time) {return new Promise(resolve => setTimeout(resolve, time))},
      deleteDraws: function() {
        //console.log('deleteDraws:',this.features)
        this.features = []
      },
      /*initLineStringDraw: function(coordinate) {
        console.log('Pegamos o primeiro clique...', coordinate)

        var event = {}

        event.coordinate = coordinate;// set your starting coordinate
        this.$refs.interactionDraw.startDrawing_(event);// tell your interaction to start drawing
        //this.features = [' { "type": "Feature", "id": "0fd25b34-9f67-4181-a18a-4d67c3882b6b", "geometry": { "type": "LineString", "coordinates": [ ' + coordinate + ' ] }, "properties": null } ']
        //this.features = [`{ "type": "Feature", "id": "0fd25b34-9f67-4181-a18a-4d67c3882b6b", "geometry": { "type": "LineString", "coordinates": [ ${coordinate} ] }, "properties": null } `]
      },*/
      zoomToLimit: function() {
        this.$refs.mapView.$view.fit(
          this.$refs.limit[0].$source.getExtent(),
          {
            size: this.$refs.map.$map.getSize(),
            duration: 1000,
          },
        )
      },
      geometryTypeToCmpName (type) {
        //console.log('geometryTypeToCmpName: ',list,feature,type)
        let name = 'vl-geom-' + kebabCase(type)
        console.log('name: ',name)
        return name
      },
    },
    watch: {
      features: {
        handler(val, oldVal){
          let n = val.length
          if(n>oldVal.length && n>0){
            this.$emit("map",['new_feature',val])
            console.log("Aqui será ultimo ponto automatico...") 
          }
        }
      },
      featuresSelected: {
        handler(val, oldVal){
          if(val.length!=oldVal.length){
            if(val.length>oldVal.length){this.$emit("map",['select_feature',val])}
            else{this.$emit("map",['unselect_feature',val])} 
          }
        }
      }
    }
  }
</script>