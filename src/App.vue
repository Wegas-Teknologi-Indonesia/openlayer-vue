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
import Overlay from 'ol/Overlay';
import LayerSwitcher from "ol-layerswitcher";
import axios from "axios";
import LayerGroup from "ol/layer/Group";
import SourceStamen from "ol/source/Stamen";

import "ol/ol.css";
import "ol-layerswitcher/dist/ol-layerswitcher.css";
import TileLayer from "ol/layer/Tile";

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

const ProvinsiWMS = new TileWMS({
  ratio: 1,
  url: "http://localhost:8080/geoserver/kemenko/wfs",
  params: {
    FORMAT: format,
    VERSION: "1.1.1",
    STYLES: "",
    LAYERS: "kemenko:prov",
    exceptions: "application/vnd.ogc.se_inimage",
  },
});

const KabupatenWMS = new TileWMS({
  ratio: 1,
  url: "http://localhost:8080/geoserver/kemenko/wfs",
  params: {
    FORMAT: format,
    VERSION: "1.1.1",
    STYLES: "",
    LAYERS: "kemenko:kabupaten",
    exceptions: "application/vnd.ogc.se_inimage",
  },
});

const untiledProvWMS = new TileLayer({
  source: ProvinsiWMS,
  visible: false,
});

const untiledKabWMS = new TileLayer({
  source: KabupatenWMS,
});

const container = document.createElement('div');
container.className = 'ol-popup';
const content = document.createElement('div');
const closer = document.createElement('div');
closer.className = 'ol-popup-closer';

container.appendChild(closer)
container.appendChild(content)

const overlay = new Overlay({
  element: container,
  autoPan: true,
  autoPanAnimation: {
    duration: 250,
  },
});

closer.onclick = function () {
  overlay.setPosition(undefined);
  closer.blur();
  return false;
};

let Maps;
const MapsView = new View({
  zoom: 0,
  center: [0, 0],
  constrainResolution: true,
});

let tempVar = {};

export default {
  name: "Openlayer Views",
  data() {
    return {
      listProvinsi: [],
      listKabupaten: [],
      dataCurrentKab: tempVar,
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
      overlays: [overlay],
      view: MapsView,
    });

    Maps.addControl(layerSwitcher);


    Maps.on("singleclick", async (evt) => {
      var view = Maps.getView();
      var viewResolution = MapsView.getResolution();
      var url = KabupatenWMS.getFeatureInfoUrl(
        evt.coordinate,
        viewResolution,
        MapsView.getProjection(),
        { INFO_FORMAT: "application/json", FEATURE_COUNT: 50 }
      );

      if (url) {
        const { data } = await axios.get(url)
        this.dataCurrentKab = data.features[0].properties;
          // alert(JSON.stringify(res.data.features[0].properties));
        tempVar = data.features[0].properties
        const datas = data.features[0].properties
        content.innerHTML = `<p>Kabupaten: ${datas.kabkot}</p><p>Kode Kabupaten: ${datas.kabkotno}</p><p>Provinsi: ${datas.provinsi}</p><p>Kode Provinsi: ${datas.provno}</p>`;
        overlay.setPosition(evt.coordinate);
      }
    });

    axios.get("http://localhost:8000/api/provinsi").then(({ data }) => {
      this.listProvinsi = data;
    });
  },
  methods: {
    async getSelectedProvinsi(event) {
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

      console.log(container);
    },
  },
};
</script>

<template>
  <div class="flex flex-row w-full h-screen">
    <div
      class="w-full h-full max-h-screen overflow-auto max-w-sm bg-gray-200 p-8"
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
        <div class="bg-white rounded-md p-4">
          <div class="flex flex-col gap-4">
            <label for="Provinsi" class="font-bold text-xl tracking-wide"
              >Properti</label
            >
            <p>Kabupaten: {{ dataCurrentKab.kabkot }}</p>
            <p>Kode Kabupaten: {{ dataCurrentKab.kabkotno }}</p>
            <p>Provinsi: {{ dataCurrentKab.provinsi }}</p>
            <p>Kode Provinsi: {{ dataCurrentKab.provno }}</p>
          </div>
        </div>
      </div>
    </div>
    <div ref="map" class="w-full h-full"></div>
  </div>
  
</template>

<style>
.ol-popup {
  position: absolute;
  background-color: white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
  padding: 15px;
  border-radius: 10px;
  border: 1px solid #cccccc;
  bottom: 12px;
  left: -50px;
  min-width: 280px;
}
.ol-popup:after,
.ol-popup:before {
  top: 100%;
  border: solid transparent;
  content: " ";
  height: 0;
  width: 0;
  position: absolute;
  pointer-events: none;
}
.ol-popup:after {
  border-top-color: white;
  border-width: 10px;
  left: 48px;
  margin-left: -10px;
}
.ol-popup:before {
  border-top-color: #cccccc;
  border-width: 11px;
  left: 48px;
  margin-left: -11px;
}
.ol-popup-closer {
  text-decoration: none;
  position: absolute;
  top: 2px;
  right: 8px;
}
.ol-popup-closer:after {
  content: "✖";
}
</style>
