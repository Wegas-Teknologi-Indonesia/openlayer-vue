<script>
import View from "ol/View";
import Map from "ol/Map";
import LayerTile from "ol/layer/Tile";
import TileWMS from "ol/source/TileWMS";
import ImageLayer from "ol/layer/Image";
import OSM from "ol/source/OSM";
import XYZ from "ol/source/XYZ";
import ImageWMS from "ol/source/ImageWMS";
import Projection from "ol/proj/Projection";
import LayerSwitcher from "ol-layerswitcher";
import axios from "axios";
import LayerGroup from "ol/layer/Group";
import SourceStamen from "ol/source/Stamen";

import "ol/ol.css";
import "ol-layerswitcher/dist/ol-layerswitcher.css";

const format = "image/png";

const osm = new LayerTile({
  title: "OSM",
  type: "base",
  visible: true,
  source: new OSM(),
});

const watercolor = new LayerTile({
  title: "Water color",
  type: "base",
  visible: false,
  source: new SourceStamen({
    layer: "watercolor",
  }),
});

const esri = new LayerTile({
  title: "ESRI Topo Map",
  type: "base",
  visible: false,
  source: new XYZ({
    attributions:
      'Tiles © <a href="https://services.arcgisonline.com/ArcGIS/' +
      'rest/services/World_Topo_Map/MapServer">ArcGIS</a>',
    url:
      "https://server.arcgisonline.com/ArcGIS/rest/services/" +
      "World_Topo_Map/MapServer/tile/{z}/{y}/{x}",
  }),
});

const baseMaps = new LayerGroup({
  title: "Base maps",
  layers: [osm, esri],
});

const layerSwitcher = new LayerSwitcher({
  reverse: true,
  groupSelectStyle: "group",
});

const ProvinsiWMS = new ImageWMS({
  ratio: 1,
  url: "http://localhost:8080/geoserver/kemenko/wms",
  params: {
    FORMAT: format,
    VERSION: "1.1.1",
    STYLES: "",
    LAYERS: "kemenko:prov",
    exceptions: "application/vnd.ogc.se_inimage",
  },
});

const KabupatenWMS = new ImageWMS({
  ratio: 1,
  url: "http://localhost:8080/geoserver/kemenko/wms",
  params: {
    FORMAT: format,
    VERSION: "1.1.1",
    STYLES: "",
    LAYERS: "kemenko:kabupaten",
    exceptions: "application/vnd.ogc.se_inimage",
  },
});

const untiledProvWMS = new ImageLayer({
  source: ProvinsiWMS,
  visible: false,
});

const untiledKabWMS = new ImageLayer({
  source: KabupatenWMS,
});

let Maps;

export default {
  name: "Openlayer Views",
  data() {
    return {
      listProvinsi: [],
      listKabupaten: [],
      selectedProvinsi: 0,
      selectedKabupaten: 0,
    };
  },
  mounted() {
    Maps = new Map({
      target: this.$refs["map"],
      layers: [
        esri,
        osm,
        untiledKabWMS,
        // untiledProvWMS,
      ],
      view: new View({
        zoom: 0,
        center: [0, 0],
        constrainResolution: true,
      }),
    });

    Maps.addControl(layerSwitcher);

    axios.get("http://localhost:8000/api/provinsi").then(({ data }) => {
      this.listProvinsi = data;
    });
  },
  methods: {
    async getSelectedProvinsi(event) {
      console.log(this.selectedProvinsi);
      const { data } = await axios.post("http://localhost:8000/api/kabupaten", {
        kode_provinsi: this.selectedProvinsi,
      });
      this.listKabupaten = data;
      this.getNewWMS();
    },
    async getNewWMS() {
      ProvinsiWMS.updateParams({
        cql_filter: "kode = " + this.selectedProvinsi,
      });
      KabupatenWMS.updateParams({
        cql_filter:
          `provno = ${this.selectedProvinsi}` +
          (this.selectedKabupaten == 0
            ? ""
            : `AND kabkotno = ${this.selectedKabupaten}`),
      });
    },
  },
};
</script>

<template>
  <div class="flex flex-row w-full h-screen">
    <div
      class="w-full h-full max-h-screen overflow-auto max-w-sm bg-gray-400 p-8"
    >
      <div class="flex flex-col gap-4 w-full h-full">
        <div class="bg-white rounded-md p-4">
          <div class="flex flex-col gap-4">
            <label for="Provinsi" class="font-bold text-xl tracking-wide"
              >Provinsi</label
            >
            <select
              v-model="selectedProvinsi"
              @change="getSelectedProvinsi"
              id="Provinsi"
              class="p-2 border"
            >
              <option value="0" selected>Pilih Provinsi</option>
              <option
                v-bind:value="provinsi.kode"
                v-for="provinsi in listProvinsi"
                :key="provinsi.name_1"
              >
                {{ provinsi.name_1 }}
              </option>
            </select>
          </div>
        </div>
        <div class="bg-white rounded-md p-4">
          <div class="flex flex-col gap-4">
            <label for="Provinsi" class="font-bold text-xl tracking-wide"
              >Kabupaten</label
            >

            <select
              v-model="selectedKabupaten"
              @change="getSelectedProvinsi"
              id="Provinsi"
              class="p-2 border"
            >
              <option value="0" selected>Pilih Kabupaten</option>
              <option
                v-bind:value="kabupaten.kabkotno"
                v-for="kabupaten in listKabupaten"
                :key="kabupaten.kabkot"
              >
                {{ kabupaten.kabkot }}
              </option>
            </select>
          </div>
        </div>
      </div>
    </div>
    <div ref="map" class="w-full h-full"></div>
  </div>
</template>

<style></style>
