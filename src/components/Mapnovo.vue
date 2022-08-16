<template>
  <div class="map" @click="autoRect">
    
    <VlMap
      v-show="mapVisible"
      ref="map"
      data-projection="EPSG:4326"
      @click="clickCoordinate = $event.coordinate"
      @created="onMapCreated"
      @dl:postrender="onMapDefLayerPostRender">
      <VlView
        ref="view"
        :center.sync="center"
        :zoom.sync="zoom"
        :rotation.sync="rotation"/>

      <!-- base layers -->
      <VlLayerTile
        v-for="layer in baseLayers"
        :key="layer.name"
        :id="layer.name"
        :visible="layer.visible">
        <component
          :is="'vl-source-' + layer.name"
          v-bind="layer"/>
      </VlLayerTile>
      <!--// base layers -->

      <!-- other layers from config -->
      <component
        v-for="layer in layers"
        :is="layer.cmp"
        :key="layer.id"
        v-bind="layer">
        <!-- add vl-source-* -->
        <component
          :is="layer.source.cmp"
          v-bind="layer.source">
          <!-- add inner source if provided (like vl-source-vector inside vl-source-cluster) -->
          <component
            v-if="layer.source.source"
            :is="layer.source.source.cmp"
            v-bind="layer.source.source">
            <!-- add static features to vl-source-vector if provided -->
            <template v-if="layer.source.source.staticFeatures && layer.source.source.staticFeatures.length">
              <VlFeature
                v-for="feature in layer.source.source.staticFeatures" :key="feature.id"
                :id="feature.id" :properties="feature.properties">
                <component
                  :is="geometryTypeToCmpName(feature.geometry.type)"
                  v-bind="feature.geometry"/>
              </VlFeature>
            </template>
          </component>
        </component>
        <!--// vl-source-* -->

        <!-- add style components if provided -->
        <!-- create vl-style or vl-style-func -->
        <template v-if="layer.style">
          <component
            v-for="(style, i) in layer.style"
            :key="i"
            :is="style.cmp"
            v-bind="style">
            <!-- create inner style components: vl-style-circle, vl-style-icon, vl-style-fill, vl-style-stroke & etc -->
            <template v-if="style.styles">
              <component
                v-for="(st, cmp) in style.styles"
                :key="cmp"
                :is="cmp"
                v-bind="st">
                <!-- vl-style-fill, vl-style-stroke if provided -->
                <VlStyleFill
                  v-if="st.fill"
                  v-bind="st.fill"/>
                <VlStyleStroke
                  v-if="st.stroke"
                  v-bind="st.stroke"/>
              </component>
            </template>
          </component>
        </template>
        <!--// style -->
      </component>
      <!--// other layers -->

      <!-- draw target -->
      <VlLayerVector
        id="draw-pane"
        :visible="mapPanel.tab === 'draw'">
        <VlSourceVector
          ident="draw-target"
          :features.sync="drawnFeatures"/>
      </VlLayerVector>

      <!-- interactions -->
      <VlInteractionSelect
        :active="mapPanel.tab !== 'draw'"
        :features.sync="selectedFeatures">
        <template slot-scope="selection">
          <!-- select styles -->
          <VlStyle>
            <VlStyleStroke
              color="#423e9e"
              :width="7"/>
            <VlStyleFill
              :color="[254, 178, 76, 0.7]"/>
            <VlStyleCircle :radius="5">
              <VlStyleStroke
                color="#423e9e"
                :width="7"/>
              <VlStyleFill
                :color="[254, 178, 76, 0.7]"/>
            </VlStyleCircle>
          </VlStyle>
          <VlStyle :z-index="1">
            <VlStyleStroke
              color="#d43f45"
              :width="2"/>
            <VlStyleCircle :radius="5">
              <VlStyleStroke
                color="#d43f45"
                :width="2"/>
            </VlStyleCircle>
          </VlStyle>
          <!--// select styles -->

          <!-- selected feature popup -->
          <VlOverlay
            v-for="feature in selection.features"
            :key="feature.id"
            :id="feature.id"
            class="feature-popup"
            :position="pointOnSurface(feature.geometry)"
            :auto-pan="true"
            :auto-pan-animation="{ duration: 300 }">
            <template #default="popup">
              <section class="card">
                <header class="card-header">
                  <p class="card-header-title">
                    Feature ID {{ feature.id }}
                  </p>
                  <a class="card-header-icon" title="Close"
                     @click="selectedFeatures = selectedFeatures.filter(f => f.id !== feature.id)">
                    <b-icon icon="close"></b-icon>
                  </a>
                </header>
                <div class="card-content">
                  <div class="content">
                    <p>
                      Overlay popup content for Feature with ID <strong>{{ feature.id }}</strong>
                    </p>
                    <p>
                      Popup: {{ JSON.stringify(popup) }}
                    </p>
                    <p>
                      Feature: {{ JSON.stringify({id: feature.id, properties: feature.properties}) }}
                    </p>
                  </div>
                </div>
              </section>
            </template>
          </VlOverlay>
          <!--// selected popup -->
        </template>
      </VlInteractionSelect>

      <VlInteractionDraw
        :active="mapPanel.tab === 'draw'"
        source="draw-target"
        :type="drawType"/>
      <VlInteractionModify
        :active="mapPanel.tab === 'draw'"
        source="draw-target"/>
      <VlInteractionSnap
        :active="mapPanel.tab === 'draw'"
        source="draw-target"
        :priority="10"/>
      <!--// interactions -->
      <vl-feature>
        <!-- <vl-geom-polygon
        :features.sync="drawnFeatures"/> -->
      </vl-feature>
    </VlMap>

    <!-- map panel, controls -->
    <div class="map-panel">
      <b-collapse class="panel box is-paddingless" :open.sync="panelOpen">
        <div slot="trigger" class="panel-heading">
          <div class="columns">
            <div class="column is-11">
              <strong>Map panel</strong>
            </div>
            <div class="column">
              <b-icon :icon="panelOpen ? 'chevron-up' : 'chevron-down'" size="is-small"></b-icon>
            </div>
          </div>
        </div>
        <p class="panel-tabs">
          <a @click="showMapPanelTab('state')" :class="mapPanelTabClasses('state')">State</a>
          <a @click="showMapPanelTab('layers')" :class="mapPanelTabClasses('layers')">Layers</a>
          <a @click="showMapPanelTab('draw')" :class="mapPanelTabClasses('draw')">Draw</a>
        </p>

        <div class="panel-block" v-show="mapPanel.tab === 'state'">
          <table class="table is-fullwidth">
            <button @click="autoRect">ok</button>
            <tr>
              <th>Map center</th>
              <td @change="autoRect">{{ center }}</td>
              
              
            </tr>
            <tr>
              <th>Map zoom</th>
              <td>{{ zoom }}</td>
            </tr>
            <tr>
              <th>Map rotation</th>
              <td>{{ rotation }}</td>
            </tr>
            <tr>
              <th>Device coordinate</th>
              <td>{{ deviceCoordinate }}</td>
            </tr>
            <tr>
              <th>Selected features</th>
              <td>{{ selectedFeatures.map(f => f.id) }}</td>
            </tr>
          </table>
        </div>

        <div class="panel-block" v-for="layer in layers" :key="layer.id" @click="showMapPanelLayer"
             :class="{ 'is-active': layer.visible }"
             v-show="mapPanel.tab === 'layers'">
          <b-switch :key="layer.id" v-model="layer.visible">
            {{ layer.title }}
          </b-switch>
        </div>

        <div style="z-index: 999;" class="panel-block draw-panel" v-show="mapPanel.tab === 'draw'">
          <p>{{drawType}}</p>
          <div class="buttons">
            <button @click="drawType = drawControls[0].type" :class="drawType && drawType === drawControls[0].type ? 'is-info' : ''" class="button"><b-icon :icon="drawControls.icon"></b-icon>{{ drawControls[0].label }}</button>
            <button @click="drawType = drawControls[1].type" :class="drawType && drawType === drawControls[1].type ? 'is-info' : ''" class="button"><b-icon :icon="drawControls.icon"></b-icon>{{ drawControls[1].label }}</button>
            <button @click="drawType = drawControls[2].type" :class="drawType && drawType === drawControls[2].type ? 'is-info' : ''" class="button"><b-icon :icon="drawControls.icon"></b-icon>{{ drawControls[2].label }}</button>
            <button @click="drawType = drawControls[3].type" :class="drawType && drawType === drawControls[3].type ? 'is-info' : ''" class="button"><b-icon :icon="drawControls.icon"></b-icon>{{ drawControls[3].label }}</button>
            <button @click="drawType = drawControls[4].type" :class="drawType && drawType === drawControls[4].type ? 'is-info' : ''" class="button"><b-icon :icon="drawControls.icon"></b-icon>{{ drawControls[4].label }}</button>

            <!-- <button v-for="control in drawControls" :key="control.type || -1" @click="drawType = control.type"
                    :class="drawType && drawType === control.type ? 'is-info' : ''" class="button" >
              
              <span>{{ control.label }}</span>
            </button> -->
          </div>
        </div>
      </b-collapse>
    </div>
    <!--// map panel, controls -->

    <!-- base layers switch -->
    <div class="base-layers-panel">
      <div class="buttons has-addons">
        <button class="button is-light" v-for="layer in baseLayers"
                :key="layer.name" :class="{ 'is-info': layer.visible }"
                @click="showBaseLayer(layer.name)">
          {{ layer.title }}
        </button>
        <button class="button is-light" @click="mapVisible = !mapVisible">
          {{ mapVisible ? 'Hide map' : 'Show map' }}
        </button>
      </div>
    </div>
    <!--// base layers -->
  {{drawnFeatures}}
  </div>
</template>

<script>


var teste=0
console.log(teste)

import {camelCase, kebabCase, random, range} from 'lodash'

import {FullScreen, ScaleLine, ZoomSlider} from 'ol/control'
import {addProjection, Projection} from 'ol/proj'

import {bbox} from 'ol/loadingstrategy'
import {getVectorContext} from 'ol/render'
import {createStyle, findPointOnSurface} from 'vuelayers/dist/ol-ext'

// Custom projection for static Image layer
let x = 1024 * 10000
let y = 968 * 10000
let imageExtent = [-x / 2, -y / 2, x / 2, y / 2]
let customProj = new Projection({
  code: 'xkcd-image',
  units: 'pixels',
  extent: imageExtent,
})
// add to the list of known projections
// after that it can be used by code
addProjection(customProj)

const easeInOut = t => 1 - Math.pow(1 - t, 3)







export default {
  name: 'Map',
  data () {
    return {
      center: [0, 0],
      
      zoom: 2,
      rotation: 0,
      clickCoordinate: undefined,
      selectedFeatures: [],
      deviceCoordinate: undefined,
      mapPanel: {
        tab: 'state',
      },
      panelOpen: true,
      mapVisible: true,
      drawControls: [
        {
          type: 'point',
          label: 'Draw Point',
          icon: 'map-marker',
        },
        {
          type: 'line-string',
          label: 'Draw LineString',
          icon: 'minus',
        },
        {
          type: 'polygon',
          label: 'Draw Polygon',
          icon: 'square-o',
        },
        {
          type: 'circle',
          label: 'Draw Circle',
          icon: 'circle-thin',
        },
        {
          type: 'autorect',
          label: 'Stop drawing',
          icon: 'times',
        },
      ],
      drawType: 'point',
      drawnFeatures: [
        ],
      // base layers
      baseLayers: [
        {
          name: 'osm',
          title: 'OpenStreetMap',
          visible: true,
        }
      ],
      // layers config
      layers: [
        // Packman vector layer with static vector features
        // rendered through vl-feature component.
        
        // Circles
        
        // Countries vector layer
        // loads GeoJSON data from remote server
        {
          id: 'countries',
          title: 'Countries',
          cmp: 'vl-layer-vector',
          visible: false,
          source: {
            cmp: 'vl-source-vector',
            url: 'https://openlayers.org/en/latest/examples/data/geojson/countries.geojson',
          },
          style: [
            {
              cmp: 'vl-style',
              styles: {
                'vl-style-fill': {
                  color: [255, 255, 255, 0.5],
                },
                'vl-style-stroke': {
                  color: '#219e46',
                  width: 2,
                },
                'vl-style-text': {
                  text: '\uf041',
                  font: '24px FontAwesome',
                  fill: {
                    color: '#2355af',
                  },
                  stroke: {
                    color: 'white',
                  },
                },
              },
            },
          ],
        },
        // Tile layer with WMS source
        {
          id: 'wms',
          title: 'WMS',
          cmp: 'vl-layer-tile',
          visible: false,
          source: {
            cmp: 'vl-source-tile-wms',
            url: 'https://ahocevar.com/geoserver/wms',
            layers: 'topp:states',
            extParams: {TILED: true},
            serverType: 'geoserver',
          },
        },
        // Tile layer with WMTS source
        {
          id: 'wmts',
          title: 'WMTS',
          cmp: 'vl-layer-tile',
          visible: false,
          source: {
            cmp: 'vl-source-wmts',
            url: 'https://services.arcgisonline.com/arcgis/rest/services/Demographics/USA_Population_Density/MapServer/WMTS/',
            layerName: '0',
            matrixSet: 'EPSG:3857',
            format: 'image/png',
            styleName: 'default',
          },
        },
        // Vector layer with clustering
        {
          id: 'cluster',
          title: 'Cluster',
          cmp: 'vl-layer-vector-image',
          visible: false,
          // Cluster source (vl-source-cluster) wraps vector source (vl-source-vector)
          source: {
            cmp: 'vl-source-cluster',
            distance: 50,
            source: {
              cmp: 'vl-source-vector',
              // features defined as array of GeoJSON encoded Features
              // to not overload Vue and DOM
              features: range(0, 10000).map(i => {
                let coordinate = [
                  random(-50, 50),
                  random(-50, 50),
                ]
                return {
                  type: 'Feature',
                  id: 'random-' + i,
                  geometry: {
                    type: 'Point',
                    coordinates: coordinate,
                  },
                }
              }),
            },
          },
          style: [
            {
              cmp: 'vl-style-func',
              function: this.clusterStyleFunc(),
            },
          ],
        },
        {
          id: 'wfs',
          title: 'WFS (Canada water areas)',
          cmp: 'vl-layer-vector-image',
          visible: false,
          source: {
            cmp: 'vl-source-vector',
            features: [],
            url(extent, resolution, projection) {
              return 'https://ahocevar.com/geoserver/wfs?service=WFS&' +
                'version=1.1.0&request=GetFeature&typename=osm:water_areas&' +
                'outputFormat=application/json&srsname=' + projection + '&' +
                'bbox=' + extent.join(',') + ',' + projection
            },
            loadingStrategy: bbox,
          },
        },
        {
          id: 'static-image',
          title: 'Static Image with custom projection',
          cmp: 'vl-layer-image',
          visible: false,
          source: {
            cmp: 'vl-source-image-static',
            projection: customProj.getCode(),
            url: 'https://imgs.xkcd.com/comics/online_communities.png',
            imageSize: [1024, 968],
            imageExtent: imageExtent,
          },
        },
        {
          id: 'wms-image',
          title: 'Image WMS',
          cmp: 'vl-layer-image',
          visible: false,
          source: {
            cmp: 'vl-source-image-wms',
            url: 'https://ahocevar.com/geoserver/wms',
            layers: 'topp:states',
            serverType: 'geoserver',
          },
        },
        {
          id: 'vector-tiles',
          title: 'Vector tiles',
          cmp: 'vl-layer-vector-tile',
          visible: false,
          source: {
            cmp: 'vl-source-vector-tile',
            url: 'https://basemaps.arcgis.com/v1/arcgis/rest/services/World_Basemap/VectorTileServer/tile/{z}/{y}/{x}.pbf',
          },
          style: [
            {
              cmp: 'vl-style',
              styles: {
                'vl-style-stroke': {
                  width: 2,
                  color: '#2979ff',
                },
                'vl-style-circle': {
                  radius: 5,
                  stroke: {
                    width: 1.5,
                    color: '#2979ff',
                  },
                },
              },
            },
          ],
        },
      ],
    }
  },
  methods: {
    camelCase,
    pointOnSurface: findPointOnSurface,
    autoRect(){
      if(this.drawType == 'autorect'){
        console.log(this.drawType)
      if(this.zoom>= 10){
        this.drawnFeatures.push({ "type": "Feature", "geometry": { "type": "Polygon", "coordinates": [ [ 
            [ ((this.center[0]).toFixed(2)*1)-(1/2/(this.zoom)),((this.center[1]).toFixed(2)*1)+(1/2/(this.zoom))], 
            [ ((this.center[0]).toFixed(2)*1)+(1/2/(this.zoom)),((this.center[1]).toFixed(2)*1)+(1/2/(this.zoom)) ], 
            [ ((this.center[0]).toFixed(2)*1)+(1/2/(this.zoom)),((this.center[1]).toFixed(2)*1)-(1/2/(this.zoom)) ], 
            [ ((this.center[0]).toFixed(2)*1)-(1/2/(this.zoom)),((this.center[1]).toFixed(2)*1)-(1/2/(this.zoom)) ] 
            ] ] }, "properties": null, "id": "4fe0a1c1-7014-4183-ace4-36dc6c841090" 
          });
      }else if(this.zoom>8){
        this.drawnFeatures.push({ "type": "Feature", "geometry": { "type": "Polygon", "coordinates": [ [ 
            [ ((this.center[0]).toFixed(2)*1)-(1*5/(this.zoom)),((this.center[1]).toFixed(2)*1)+(1*5/(this.zoom))], 
            [ ((this.center[0]).toFixed(2)*1)+(1*5/(this.zoom)),((this.center[1]).toFixed(2)*1)+(1*5/(this.zoom)) ], 
            [ ((this.center[0]).toFixed(2)*1)+(1*5/(this.zoom)),((this.center[1]).toFixed(2)*1)-(1*5/(this.zoom)) ], 
            [ ((this.center[0]).toFixed(2)*1)-(1*5/(this.zoom)),((this.center[1]).toFixed(2)*1)-(1*5/(this.zoom)) ] 
            ] ] }, "properties": null, "id": "4fe0a1c1-7014-4183-ace4-36dc6c841090" 
          });
      }else if(this.zoom>4){
        this.drawnFeatures.push({ "type": "Feature", "geometry": { "type": "Polygon", "coordinates": [ [ 
            [ ((this.center[0]).toFixed(2)*1)-(1*10/(this.zoom)),((this.center[1]).toFixed(2)*1)+(1*10/(this.zoom))], 
            [ ((this.center[0]).toFixed(2)*1)+(1*10/(this.zoom)),((this.center[1]).toFixed(2)*1)+(1*10/(this.zoom)) ], 
            [ ((this.center[0]).toFixed(2)*1)+(1*10/(this.zoom)),((this.center[1]).toFixed(2)*1)-(1*10/(this.zoom)) ], 
            [ ((this.center[0]).toFixed(2)*1)-(1*10/(this.zoom)),((this.center[1]).toFixed(2)*1)-(1*10/(this.zoom)) ] 
            ] ] }, "properties": null, "id": "4fe0a1c1-7014-4183-ace4-36dc6c841090" 
          });
      }else if(this.zoom>2){
        this.drawnFeatures.push({ "type": "Feature", "geometry": { "type": "Polygon", "coordinates": [ [ 
            [ ((this.center[0]).toFixed(2)*1)-(1*20/(this.zoom)),((this.center[1]).toFixed(2)*1)+(1*20/(this.zoom))], 
            [ ((this.center[0]).toFixed(2)*1)+(1*20/(this.zoom)),((this.center[1]).toFixed(2)*1)+(1*20/(this.zoom)) ], 
            [ ((this.center[0]).toFixed(2)*1)+(1*20/(this.zoom)),((this.center[1]).toFixed(2)*1)-(1*20/(this.zoom)) ], 
            [ ((this.center[0]).toFixed(2)*1)-(1*20/(this.zoom)),((this.center[1]).toFixed(2)*1)-(1*20/(this.zoom)) ] 
            ] ] }, "properties": null, "id": "4fe0a1c1-7014-4183-ace4-36dc6c841090" 
          });
      }
      }
    },
    geometryTypeToCmpName(type) {
      return 'vl-geom-' + kebabCase(type)
    },

    /**
     * Cluster layer style function factory
     * @return {module:ol/style/Style~StyleFunction}
     */
    clusterStyleFunc() {
      const cache = {}
      return function __clusterStyleFunc(feature) {
        const size = feature.get('features').length
        let style = cache[size]
        if (!style) {
          style = createStyle({
            imageRadius: 10,
            imageStrokeColor: '#fff',
            imageFillColor: '#3399cc',
            text: size.toString(),
            textFillColor: '#fff',
          })
          // with OpenLayers native API
          // import { Style, Circle, Fill, Stroke, Text } from 'ol/style'
          // style = new Style({
          //   image: new Circle({
          //     radius: 10,
          //     fill: new Fill({ color: '#3399cc' }),
          //     stroke: new Stroke({ color: '#fff' }),
          //   }),
          //   text: new Text({
          //     text: size.toString(),
          //     fill: new Fill({ color: '#fff' }),
          //   }),
          // })
          cache[size] = style
        }
        return [style]
      }
    },
    selectFilter(feature) {
      return ['position-feature'].indexOf(feature.getId()) === -1
    },
    onUpdatePosition(coordinate) {
      this.deviceCoordinate = coordinate
    },
    onMapDefLayerPostRender(evt) {
      // animate the marker feature from vl-map default layer
      if (!this.$refs.marker || !this.$refs.marker.$feature) return

      const feature = this.$refs.marker.$feature
      if (!feature.getGeometry() || !feature.getStyle()) return

      const frameState = evt.frameState
      const vectorContext = getVectorContext(evt)
      const flashGeom = feature.getGeometry().clone()
      const duration = feature.get('duration')
      const elapsed = frameState.time - feature.get('start')
      const elapsedRatio = elapsed / duration
      const radius = easeInOut(elapsedRatio) * 35 + 5
      const opacity = easeInOut(1 - elapsedRatio)
      const fillOpacity = easeInOut(0.5 - elapsedRatio)
      vectorContext.setStyle(createStyle({
        imageRadius: radius,
        imageFillColor: [119, 170, 203, fillOpacity],
        imageStrokeColor: [119, 170, 203, opacity],
        imageStrokeWidth: 2 + opacity,
      }))
      vectorContext.drawGeometry(flashGeom)
      if (elapsed > duration) {
        feature.set('start', Date.now())
      }

      this.$refs.map.render()
    },
    onMapCreated(vm) {
      // now ol.Map instance is ready and we can work with it directly
      console.log('map vm', vm)
      vm.addControls([
        new ScaleLine(),
        new FullScreen(),
        // TODO need to provide some base layers
        // new OverviewMap({
        //   collapsed: false,
        //   collapsible: true,
        // }),
        new ZoomSlider(),
      ])
    },
    // base layers
    showBaseLayer(name) {
      let layer = this.baseLayers.find(layer => layer.visible)
      if (layer != null) {
        layer.visible = false
      }
      layer = this.baseLayers.find(layer => layer.name === name)
      if (layer != null) {
        layer.visible = true
      }
    },
    // map panel
    mapPanelTabClasses(tab) {
      return {
        'is-active': this.mapPanel.tab === tab,
      }
    },
    showMapPanelLayer(layer) {
      layer.visible = !layer.visible
    },
    showMapPanelTab(tab) {
      this.mapPanel.tab = tab
    },
  },
}
</script>

<style lang="scss" scoped>
@import "~bulma/sass/utilities/all";

.map {
  position: relative;
  width: 100%;
  height: 100%;

  .vl-map {
    position: relative;
  }
}

.map-panel {
  padding: 0;

  .panel-heading {
    box-shadow: 0 .25em .5em transparentize($dark, 0.8);
  }

  .panel-content {
    background: $white;
    box-shadow: 0 .25em .5em transparentize($dark, 0.8);
  }

  .panel-block {
    &.draw-panel {
      .buttons {
        .button {
          display: block;
          flex: 1 1 100%;
        }
      }
    }
  }

  @include tablet() {
    position: absolute;
    top: 0;
    right: 0;
    max-height: 500px;
    width: 22em;
  }
}

.base-layers-panel {
  position: absolute;
  left: 50%;
  bottom: 20px;
  transform: translateX(-50%);
}

.feature-popup {
  position: absolute;
  left: -50px;
  bottom: 12px;
  width: 20em;
  max-width: none;

  &:after, &:before {
    top: 100%;
    border: solid transparent;
    content: ' ';
    height: 0;
    width: 0;
    position: absolute;
    pointer-events: none;
  }

  &:after {
    border-top-color: white;
    border-width: 10px;
    left: 48px;
    margin-left: -10px;
  }

  &:before {
    border-top-color: #cccccc;
    border-width: 11px;
    left: 48px;
    margin-left: -11px;
  }

  .card-content {
    max-height: 20em;
    overflow: auto;
  }

  .content {
    word-break: break-all;
  }
}
</style>
