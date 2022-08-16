<template>
<div>
    <div>
        <ul>
            <li @click="drawType='point'">point</li>
            <li @click="drawType='line-string'">line</li>
            <li @click="drawType='polygon'">polygon</li>
            <li @click="edit">edit</li>
        </ul>
    </div>
    <Map
        ref="mapComponent"
        :baseLayer="baseLayers[4]"
        :layers="layers"
        :drawType="drawType"
        @map="mapEvent($event)"
    />
</div>
  
</template>

<script>
import Map from "./Map.vue";
export default {
  name: "Project",
  components: { Map },
  data() {
    return {
      baseLayers: [
        {
          name: "xyz",
          title: "Open Topo Map",
          //attributions: '<a href="https://www.google.at/permissions/geoguidelines/attr-guide.html">Map data ©2015 Google</a>',
          url: "https://tile.opentopomap.org/{z}/{x}/{y}.png",
          visible: false,
          //projection: 'EPSG:3857',
        },
        {
          name: "xyz",
          title: "World Hillshage ArcGIS",
          //attributions: '<a href="https://www.google.at/permissions/geoguidelines/attr-guide.html">Map data ©2015 Google</a>',
          url: "http://services.arcgisonline.com/ArcGIS/rest/services/Elevation/World_Hillshade/MapServer/tile/{z}/{y}/{x}",
          visible: false,
          //projection: 'EPSG:3857',
        },
        {
          name: "xyz",
          title: "Google Aerial",
          attributions:
            '<a href="https://www.google.at/permissions/geoguidelines/attr-guide.html">Map data ©2015 Google</a>',
          url: "https://mt1.google.com/vt/lyrs=s&x={x}&y={y}&z={z}",
          visible: false,
          //projection: 'EPSG:3857',
        },
        {
          name: "bingmaps",
          title: "Bing Aerial With Labels",
          apiKey:
            "ArbsA9NX-AZmebC6VyXAnDqjXk6mo2wGCmeYM8EwyDaxKfQhUYyk0jtx6hX5fpMn",
          imagerySet: "AerialWithLabels", //'CanvasGray' 'Aerial'
          visible: false,
          //projection: 'EPSG:4326',
        },
        {
          name: "osm",
          title: "Open Street Map",
          visible: true,
          //projection: 'EPSG:3857',
        },
      ],
      layers: {

      },
      drawType: "line-string",
    };
  },
  methods: {
    mapEvent: async function(data){
        try{
          console.log('init mapEvent')
          data;
         }catch(err){
          this.$eventHub.$emit('alert',{type:'error', message: 'Map event error'})
          console.log('ERROR:',err.response.data)
        }
        
      },
    edit:function (){
        //this.coordinates=undefined
        this.drawType = undefined
        this.teste()
    },
    teste:function(){
        console.log(this.coordinates)
    }
  },
};
</script>
