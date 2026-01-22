<template>
  <div id="map-container">
    <div id="map">
      <center>{{ mapMessage }}</center>
      <div class="leaflet-bottom leaflet-right" v-if="canCreateAdvClaims">
        <b-button-group size="sm" id="selection-control" vertical>
          <b-button :disabled="selectionMode === 'area'" @click="areaSelect"
            >Select area</b-button
          >
          <b-button :disabled="selectionMode === 'region'" @click="regionSelect"
            >Select region</b-button
          >
          <b-button @click="clearSelection" variant="danger"
            >Clear selection</b-button
          >
        </b-button-group>
      </div>
    </div>
  </div>
</template>

<script>
import L from "leaflet";
import "leaflet.markercluster";
import { eventBus } from "../main";
import { setInterval } from "timers";
import getClaims from "../helpers/getClaims";

const playerIcon = L.icon({
  iconUrl: "img/marker-survivor.png",
  iconRetinaUrl: "img/marker-survivor-2x.png",
  iconSize: [25, 48],
  iconAnchor: [12, 24],
  popupAnchor: [0, -20],
});

const homeIcon = L.icon({
  iconUrl: "img/home-icon.png",
  iconSize: [40, 40],
  iconAnchor: [12, 24],
  popupAnchor: [0, -20],
});

const vehicleIcon = L.icon({
  iconUrl: "img/vehicle-icon.png",
  iconSize: [50, 50],
  iconAnchor: [12, 24],
  popupAnchor: [0, -20],
});

const vehicleIcons = {
  bicycle: L.icon({
    iconUrl: "img/ui_game_symbol_bicycle.png",
    iconSize: [50, 50],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
  minibike: L.icon({
    iconUrl: "img/ui_game_symbol_minibike.png",
    iconSize: [50, 50],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
  motorcycle: L.icon({
    iconUrl: "img/ui_game_symbol_motorcycle.png",
    iconSize: [50, 50],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
  gyrocopter: L.icon({
    iconUrl: "img/ui_game_symbol_gyrocopter.png",
    iconSize: [50, 50],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
  truck4x4: L.icon({
    iconUrl: "img/ui_game_symbol_4x4.png",
    iconSize: [50, 50],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
};

const traderIcon = L.icon({
  iconUrl: "img/shopping-cart.png",
  iconSize: [40, 40],
  iconAnchor: [12, 24],
  popupAnchor: [0, -20],
});

const traderIcons = {
  trader_bob: L.icon({
    iconUrl: "img/trader_bob.png",
    iconSize: [40, 40],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
  trader_hugh: L.icon({
    iconUrl: "img/trader_hugh.png",
    iconSize: [40, 40],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
  trader_jen: L.icon({
    iconUrl: "img/trader_jen.png",
    iconSize: [40, 40],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
  trader_joel: L.icon({
    iconUrl: "img/trader_joel.png",
    iconSize: [40, 40],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
  trader_rekt: L.icon({
    iconUrl: "img/trader_rekt.png",
    iconSize: [40, 40],
    iconAnchor: [12, 24],
    popupAnchor: [0, -20],
  }),
};

export default {
  name: "sdtd-map",
  data: function() {
    return {
      map: null,
      tileLayer: null,
      layers: [],
      mapMessage: "Please fill in connection info in the server settings tab.",
      mapInfo: {
        regionsize: 512,
        chunksize: 16,
        tilesize: 128,
        maxzoom: 4,
      },
      userStatus: {
        loggedin: false,
        username: null,
        permissionlevel: 2000,
        permissions: [],
      },
      clickMarkers: [],
      // Rectangles on the map
      selectedRegionsRecs: [],
      // Region names
      selectedRegions: [],
      selectedArea: null,
      claimTypes: window.claimTypes,
      connectionInfo: {
        ip: "",
        port: "",
        adminUser: "",
        adminToken: "",
      },
      selectionMode: "area",
      colors: [
        "purple",
        "cyan",
        "red",
        "green",
        "blue",
        "yellow",
        "orange",
        "brown",
        "coral",
        "chocolate",
        "skyblue",
        "#fc079e",
        "#bdd3cb",
        "#bada55",
        "#008000",
        "#ffc0cb",
      ],
      // After all claims for a type are finished drawing, this is incremented
      colorIterator: 0,
    };
  },
  computed: {
    ClaimCreatorPort: function() {
      return parseInt(this.connectionInfo.port) + 1;
    },
    activeColor: function() {
      return this.colors[this.colorIterator];
    },
    canCreateAdvClaims: function() {
      return this.hasPermission("ClaimCreator.createadvclaims");
    },
  },

  async mounted() {
    this.userStatus = this.getUserStatus();
    this.userStatus.then((status) => {
      this.userStatus = status;

      setInterval(() => {
        this.drawLandClaims();
        this.drawPlayers();
        this.drawHomes();
        this.drawQuestPoi();
        this.drawTraders();
        this.drawPois();
      }, 30000);

      this.createMap();

      eventBus.$on("refresh-claims", () => {
        if (this.map != null) {
          this.map.remove();
          this.map = null;
          this.createMap();
        }
      });
    });
  },

  methods: {
    hasPermission(permModule) {
      if (!this.userStatus || !Array.isArray(this.userStatus.permissions)) {
        return false;
      }
      const permission = this.userStatus.permissions.find(
        (p) => p.module.toUpperCase() === permModule.toUpperCase()
      );
      if(!permission) return false;

      return permission.allowed;
    },
    parseVehicleName(rawName) {
      const parts = String(rawName || "").split(/<BR>/i);
      let header = parts[0] || "";
      let status = null;
      let id = null;
      let owner = null;

      const statusMatch = header.match(/^\(([^)]+)\)/);
      if (statusMatch) {
        status = statusMatch[1].trim();
        header = header.replace(statusMatch[0], "").trim();
      }

      const idMatch = header.match(/\(Id:\s*([^)]+)\)/i);
      if (idMatch) {
        id = idMatch[1].trim();
        header = header.replace(idMatch[0], "").trim();
      }

      for (const part of parts.slice(1)) {
        const ownerMatch = part.match(/Owner:\s*(.*)/i);
        if (ownerMatch) {
          owner = ownerMatch[1].trim();
          break;
        }
      }

      return {
        vehicleName: header || rawName,
        status,
        id,
        owner,
      };
    },
    formatVehiclePopup(vehicle) {
      const parsed = this.parseVehicleName(vehicle.name);
      const lines = [];
      lines.push(`Vehicle: ${parsed.vehicleName}`);
      if (this.canCreateAdvClaims) {
        if (parsed.status) {
          lines.push(`Status: ${parsed.status}`);
        }
        if (parsed.id) {
          lines.push(`Id: ${parsed.id}`);
        }
      }
      if (parsed.owner) {
        lines.push(`Owner: ${parsed.owner}`);
      }
      if (this.canCreateAdvClaims) {
        lines.push(
          `Position: ${vehicle.posX} ${vehicle.posY} ${vehicle.posZ}`
        );
      }
      return lines.join("<br>");
    },
    getVehicleIcon(vehicleName) {
      const name = String(vehicleName || "").toLowerCase();
      if (name.includes("bicycle") || name.includes("велосипед")) {
        return vehicleIcons.bicycle;
      }
      if (
        name.includes("minibike") ||
        name.includes("mini-bike") ||
        name.includes("мини-байк") ||
        name.includes("мини байк")
      ) {
        return vehicleIcons.minibike;
      }
      if (name.includes("motorcycle") || name.includes("мотоцикл")) {
        return vehicleIcons.motorcycle;
      }
      if (name.includes("gyrocopter") || name.includes("gyro")) {
        return vehicleIcons.gyrocopter;
      }
      if (
        name.includes("truck4x4") ||
        name.includes("4x4") ||
        name.includes("truck")
      ) {
        return vehicleIcons.truck4x4;
      }
      return vehicleIcon;
    },
    getTraderIcon(traderName) {
      const baseName = String(traderName || "").split(".")[0];
      return traderIcons[baseName] || traderIcon;
    },
    formatPlayerPopup(player) {
      const playerLabel = this.formatSteamLink(player.name, player.steamid);
      if (this.canCreateAdvClaims) {
        return `${playerLabel}<br>${player.steamid}<br>Position: ${
          player.position.x
        } ${player.position.y} ${player.position.z}`;
      }
      return `${playerLabel}`;
    },
    formatTraderName(traderName) {
      if (this.canCreateAdvClaims) {
        return traderName;
      }
      const baseName = String(traderName || "").split(".")[0];
      if (!baseName.startsWith("trader_")) {
        return traderName;
      }
      const rawName = baseName.slice("trader_".length).replace(/_/g, " ");
      const title = rawName.replace(/\b\w/g, (char) => char.toUpperCase());
      return `Trader ${title}`;
    },
    formatLandClaimPopup(lcbOwner, lcb) {
      const ownerLabel = this.formatSteamLink(
        lcbOwner.playername,
        lcbOwner.steamid
      );
      if (!this.canCreateAdvClaims) {
        return `Player: ${ownerLabel}`;
      }
      return `Player: ${ownerLabel}<br>Status: ${
        lcbOwner.claimactive ? "Active" : "Inactive"
      }<br>${lcbOwner.steamid}<br>${lcbOwner.eos_id}<br>Position: ${lcb.x} ${
        lcb.y
      } ${lcb.z}`;
    },
    formatSteamLink(name, steamid) {
      const safeName = String(name || "");
      let safeId = String(steamid || "").trim();
      if (safeId.toLowerCase().startsWith("steam_")) {
        safeId = safeId.slice("steam_".length);
      }
      if (!safeId) {
        return safeName;
      }
      return `<a class="steam-profile-link" href="https://steamcommunity.com/profiles/${safeId}" target="_blank" rel="noopener noreferrer">${safeName}</a>`;
    },
    getUserStatus() {
      return fetch("/userstatus")
        .then((response) => {
          return response.json();
        })
        .then((data) => {
          return data;
        });
    },
    getLandClaims() {
      return fetch(`/api/getlandclaims`)
        .then(function(response) {
          return response.json();
        })
        .then(function(data) {
          return data;
        });
    },
    async drawLandClaims() {
      if (!this.hasPermission("ClaimCreator.getlandclaims")) {
        return;
      }

      const landClaimData = await this.getLandClaims();
      const claimRadius = (landClaimData.claimsize - 1) / 2;
      let lcbLayer = this.layers["Land claim blocks"];
      if (!lcbLayer) {
        this.layers["Land claim blocks"] = new L.LayerGroup();
        lcbLayer = this.layers["Land claim blocks"];
      }

      lcbLayer.clearLayers();
      for (const lcbOwner of landClaimData.claimowners) {
        for (const lcb of lcbOwner.claims) {
          // Create the LCB marker & area
          let claimcolor = this.canCreateAdvClaims
            ? lcbOwner.claimactive
              ? "green"
              : "red"
            : "green";
          const lcbArea = L.rectangle([
            [lcb.x - claimRadius, lcb.z - claimRadius],
            [lcb.x + claimRadius, lcb.z + claimRadius]],
            {color: claimcolor,
            weight: 1,
          }).bindPopup(this.formatLandClaimPopup(lcbOwner, lcb));
          lcbLayer.addLayer(lcbArea);
        }
      }
      return lcbLayer;
    },
    getPois() {
      return fetch(`/api/getallpois`)
        .then(function(response) {
          return response.json();
        })
        .then(function(data) {
          return data;
        });
    },
    async drawPois() {
      if (!this.hasPermission("ClaimCreator.getallpois")) {
        return;
      }

      const { AllPOIs: pois } = await this.getPois();
      let poisLayer = this.layers["POIs"];
      if (!poisLayer) {
        this.layers["POIs"] = new L.LayerGroup();
        poisLayer = this.layers["POIs"];
      }

      for (const poi of pois) {
        const poiRec = this.createClaimRectangle(
          {
            W: poi.minx,
            E: poi.maxx,
            S: poi.minz,
            N: poi.maxz,
          },
          undefined,
          "orange"
        );
        poiRec.bindPopup(
          `${poi.name} - ${poi.containsbed ? "Contains bed" : "Unclaimed"}`
        );
        poisLayer.addLayer(poiRec);
      }
    },
    getTraders() {
      return fetch(`/api/gettraders`)
        .then(function(response) {
          return response.json();
        })
        .then(function(data) {
          return data;
        });
    },
    async drawTraders() {
      if (!this.hasPermission("ClaimCreator.gettraders")) {
        return;
      }

      const { Traders } = await this.getTraders();
      let tradersLayer = this.layers["Traders"];
      if (!tradersLayer) {
        this.layers["Traders"] = new L.LayerGroup();
        tradersLayer = this.layers["Traders"];
      }

      tradersLayer.clearLayers();
      for (const trader of Traders) {
        const traderRec = this.createClaimRectangle(
          {
            W: trader.minx,
            E: trader.maxx,
            S: trader.minz,
            N: trader.maxz,
          },
          undefined,
          "green"
        );
        const marker = L.marker([trader.x, trader.z], {
          icon: this.getTraderIcon(trader.name),
        }).bindPopup(this.formatTraderName(trader.name));
        tradersLayer.addLayer(marker);
        tradersLayer.addLayer(traderRec);
      }
    },
    getPlayers() {
      return fetch(`/api/getplayersonline`)
        .then(function(response) {
          return response.json();
        })
        .then(function(data) {
          return data;
        });
    },
    async drawPlayers() {
      if (!this.hasPermission("ClaimCreator.getplayersonline")) {
        return;
      }

      const currentPlayers = await this.getPlayers();
      let playersLayer = this.layers["Online players"];
      if (!playersLayer) {
        this.layers["Online players"] = L.markerClusterGroup({
          showCoverageOnHover: false,
          spiderfyOnMaxZoom: true,
          iconCreateFunction: this.createClusterIcon("player"),
        });
        playersLayer = this.layers["Online players"];
      }

      playersLayer.clearLayers();
      for (const player of currentPlayers) {
        // Create the player marker & area
        const marker = L.marker([player.position.x, player.position.z], {
          icon: playerIcon,
        }).bindPopup(this.formatPlayerPopup(player));
        playersLayer.addLayer(marker);
      }
      return playersLayer;
    },
    getQuestPoi(filterClaim) {
      const link = `/api/getquestpois${
        filterClaim ? "?filter=bedlcbonly" : ""
      }`;
      return fetch(link)
        .then(function(response) {
          return response.json();
        })
        .then(function(data) {
          return data;
        });
    },
    async drawQuestPoi() {
      if (!this.hasPermission("ClaimCreator.getquestpois")) {
        return;
      }

      const pois = await this.getQuestPoi();

      let poisLayer = this.layers["Quest POIs"];
      if (!poisLayer) {
        this.layers["Quest POIs"] = new L.LayerGroup();
        poisLayer = this.layers["Quest POIs"];
      }

      let claimedPoisLayer = this.layers["Claimed quest POIs"];
      if (!claimedPoisLayer) {
        this.layers["Claimed quest POIs"] = new L.LayerGroup();
        claimedPoisLayer = this.layers["Claimed quest POIs"];
      }

      for (const poi of pois.QuestPOIs) {
        const poiRec = this.createClaimRectangle(
          {
            W: poi.minx,
            E: poi.maxx,
            S: poi.minz,
            N: poi.maxz,
          },
          undefined,
          poi.containsbed ? "red" : "blue"
        ).bindPopup(poi.name);

        poisLayer.addLayer(poiRec);

        if (poi.containsbed) {
          claimedPoisLayer.addLayer(poiRec);
        }
      }
    },
    getVehicles() {
      return fetch(`/api/getvehicles`)
        .then(function(response) {
          return response.json();
        })
        .then(function(data) {
          return data;
        });
    },
    async drawVehicles() {
      if (!this.hasPermission("ClaimCreator.getvehicles")) {
        return;
      }

      const currentVehicles = await this.getVehicles();
      let vehiclesLayer = this.layers["Vehicles"];
      if (!vehiclesLayer) {
        this.layers["Vehicles"] = L.markerClusterGroup({
          showCoverageOnHover: false,
          spiderfyOnMaxZoom: true,
          iconCreateFunction: this.createClusterIcon("vehicle"),
        });
        vehiclesLayer = this.layers["Vehicles"];
      }

      vehiclesLayer.clearLayers();
      for (const vehicle of currentVehicles.Vehicles) {
        const marker = L.marker([vehicle.posX, vehicle.posZ], {
          icon: this.getVehicleIcon(vehicle.name),
        }).bindPopup(this.formatVehiclePopup(vehicle));
        vehiclesLayer.addLayer(marker);
      }
    },
    getHomes() {
      return fetch(`/api/getplayerhomes`)
        .then(function(response) {
          return response.json();
        })
        .then(function(data) {
          return data;
        });
    },
    async drawHomes() {
      if (!this.hasPermission("ClaimCreator.getplayerhomes")) {
        return;
      }

      const currentHomes = await this.getHomes();
      let homesLayer = this.layers["Homes"];
      if (!homesLayer) {
        this.layers["Homes"] = new L.LayerGroup();
        homesLayer = this.layers["Homes"];
      }

      homesLayer.clearLayers();
      for (const home of currentHomes.homeowners) {
        const homeRec = this.createClaimRectangle(
          {
            W: home.x - currentHomes.homesize,
            E: home.x + currentHomes.homesize,
            S: home.z - currentHomes.homesize,
            N: home.z + currentHomes.homesize,
          },
          undefined,
          home.active ? "green" : "red"
        );

        const marker = L.marker([home.x, home.z], {
          icon: homeIcon,
        }).bindPopup(
          `Home owner: ${home.steamid} <br> EOS_id: ${home.eos_id} <br> Position: ${home.x} ${home.y} ${
            home.z
          }`
        );

        homesLayer.addLayer(homeRec);
        homesLayer.addLayer(marker);
      }
    },
    areaSelect(e) {
      // Make sure the map does not get the event, otherwise an invalid click is registered
      L.DomEvent.stop(e);
      this.clearSelection();
      this.selectionMode = "area";
    },
    regionSelect(e) {
      // Make sure the map does not get the event, otherwise an invalid click is registered
      L.DomEvent.stop(e);
      this.clearSelection();
      this.selectionMode = "region";
    },
    clearSelection(e) {
      // This function is not always called as a result of a button click
      if (e) {
        // Make sure the map does not get the event, otherwise an invalid click is registered
        L.DomEvent.stop(e);
      }
      // Area
      for (const marker of this.clickMarkers) {
        marker.remove();
      }

      this.clickMarkers = [];
      if (this.selectedArea) {
        this.selectedArea.remove();
      }

      // Regions
      for (const region of this.selectedRegionsRecs) {
        region.remove();
      }
      this.selectedRegions = [];
      eventBus.$emit("area-selected", null);
      eventBus.$emit("region-selected", this.selectedRegions);
    },
    handleRegionClick() {},
    handleAreaClick() {},
    createMap() {
      this.mapMessage = "";
      this.initMap();
      this.initLayers();
      this.createCoordinateControl();
      this.map.on("click", (event) => {
        if (!this.canCreateAdvClaims) {
          return;
        }
        if (this.selectionMode === "area") {
          if (this.clickMarkers.length > 1) {
            this.clickMarkers[0].remove();
            this.clickMarkers = this.clickMarkers.slice(
              1,
              this.clickMarkers.length
            );
          }

          this.clickMarkers.push(
            L.circle(event.latlng, { radius: 1 }).addTo(this.map)
          );

          if (this.clickMarkers.length === 2) {
            const latLngSelected = [
              this.clickMarkers[0].getLatLng(),
              this.clickMarkers[1].getLatLng(),
            ];
            if (this.selectedArea) {
              this.selectedArea.remove();
            }
            const selectedArea = L.rectangle(latLngSelected, {
              weight: 1,
            }).addTo(this.map);

            this.selectedArea = selectedArea;
            eventBus.$emit("area-selected", latLngSelected);
          }
        }

        if (this.selectionMode === "region") {
          const region = this.CoordToRegion(event.latlng);
          const regionName = this.FormatRegionFileName(region);
          const claimDetails = {
            W: region.lat * this.mapInfo.regionsize,
            E: region.lat * this.mapInfo.regionsize + this.mapInfo.regionsize,
            N: region.lng * this.mapInfo.regionsize + this.mapInfo.regionsize,
            S: region.lng * this.mapInfo.regionsize,
            Name: regionName,
            Type: "To be confirmed reset region",
          };
          const regionRec = this.createClaimRectangle(
            claimDetails,
            "Selected region",
            "grey"
          );
          regionRec.addTo(this.map);

          this.selectedRegionsRecs.push(regionRec);
          this.selectedRegions.push(regionName);
          eventBus.$emit("region-selected", this.selectedRegions);
        }
      });
    },
    GetSdtdTileLayer(mapinfo) {
      var tileLayer = L.tileLayer(`/map/{z}/{x}/{y}.png`, {
        maxZoom: mapinfo.maxzoom + 1,
        minZoom: Math.max(0, mapinfo.maxzoom - 5),
        maxNativeZoom: mapinfo.maxzoom,
        minNativeZoom: 0,
        tileSize: mapinfo.tilesize,
      });

      tileLayer.getTileUrl = function(coords) {
        coords.y = -coords.y - 1;
        return L.TileLayer.prototype.getTileUrl.bind(tileLayer)(coords);
      };

      return tileLayer;
    },
    initMap() {
      const mapInfo = this.mapInfo;
      // 7dtd coordinate transformations
      const SDTD_Projection = {
        project: function(latlng) {
          return new L.Point(
            latlng.lat / Math.pow(2, mapInfo.maxzoom),
            latlng.lng / Math.pow(2, mapInfo.maxzoom)
          );
        },

        unproject: function(point) {
          return new L.LatLng(
            point.x * Math.pow(2, mapInfo.maxzoom),
            point.y * Math.pow(2, mapInfo.maxzoom)
          );
        },
      };

      const SDTD_CRS = L.extend({}, L.CRS.Simple, {
        projection: SDTD_Projection,
        transformation: new L.Transformation(1, 0, -1, 0),

        scale: function(zoom) {
          return Math.pow(2, zoom);
        },
      });

      this.map = L.map("map", {
        attributionControl: false,
        crs: SDTD_CRS,
        zoomControl: false,
      }).setView([0, 0], 0);

      L.control
        .zoom({
          position: "topright",
        })
        .addTo(this.map);
    },
    async drawAllClaims() {
      this.colorIterator = 0;
      for (const claimType of this.claimTypes) {
        await this.drawClaim(claimType);
        this.colorIterator++;
      }
      this.colorIterator = 0;
      await this.drawClaim("resetregion");
    },
    async drawClaim(claimType) {
      if (claimType === "resetregion") {
        if (!this.hasPermission("ClaimCreator.getresetregions")) {
          return;
        }
      } else {
        if (!this.hasPermission("ClaimCreator.getadvclaims")) {
          return;
        }
      }

      const claims = await getClaims(claimType);
      const rectangles = [];
      for (const claim of claims) {
        rectangles.push(this.createClaimRectangle(claim, claimType));
      }
      this.layers[claimType] = new L.LayerGroup(rectangles);
      this.layers[claimType].addTo(this.map);
    },
    createClaimRectangle(claim, type, color) {
      const rectangle = L.rectangle([[claim.W, claim.S], [claim.E, claim.N]], {
        color: color || this.activeColor,
        weight: 1,
      });
      if (type) {
        let popup;
        if (type === "resetregion") {
          popup = L.popup().setContent(`Reset region`);
        } else {
          popup = L.popup().setContent(
            `Name: ${claim.Name} Type: ${type} ${claim.Type}`
          );
        }
        rectangle.bindPopup(popup);
      }
      return rectangle;
    },
    async initLayers() {
      this.tileLayer = this.GetSdtdTileLayer(this.mapInfo);

      this.tileLayer.addTo(this.map);
      await this.drawAllClaims();
      await this.drawLandClaims();
      await this.drawPlayers();
      await this.drawHomes();
      await this.drawVehicles();
      await this.drawQuestPoi();
      await this.drawTraders();
      await this.drawPois();
      this.layers["Regions"] = this.getRegionLayer(this.mapInfo);
      L.control.layers(null, this.layers).addTo(this.map);
    },
    FormatRegionFileName(latlng) {
      return "r." + latlng.lat + "." + latlng.lng + ".7rg";
    },
    CoordToRegion(latlng) {
      var x = Math.floor(
        (latlng.lat + 16777216) / this.mapInfo.regionsize -
          16777216 / this.mapInfo.regionsize
      );
      var y = Math.floor(
        (latlng.lng + 16777216) / this.mapInfo.regionsize -
          16777216 / this.mapInfo.regionsize
      );
      return L.latLng(x, y);
    },
    getRegionLayer(mapInfo) {
      // Copyright Alloc
      // https://7dtd.illy.bz

      // Yeah it's stupid, I know.
      // Keeps a reference to the Vue Instance
      const self = this;

      var regionLayer = L.gridLayer({
        maxZoom: mapInfo.maxzoom + 1,
        minZoom: 0,
        maxNativeZoom: mapInfo.maxzoom + 1,
        tileSize: mapInfo.tilesize,
      });

      regionLayer.createTile = function(tilePoint) {
        var blockWorldSize =
          mapInfo.tilesize * Math.pow(2, mapInfo.maxzoom - tilePoint.z);
        var tileLeft = tilePoint.x * blockWorldSize;
        var tileBottom = (-1 - tilePoint.y) * blockWorldSize;
        var blockPos = L.latLng(tileLeft, tileBottom);

        var canvas = L.DomUtil.create("canvas", "leaflet-tile");
        canvas.width = mapInfo.tilesize;
        canvas.height = mapInfo.tilesize;
        var ctx = canvas.getContext("2d");

        ctx.fillStyle = "white";
        ctx.lineWidth = 1;
        ctx.font = "14px Arial";
        ctx.strokeStyle = "black";

        var lineCount = blockWorldSize / mapInfo.regionsize;
        if (lineCount >= 1) {
          var pos = 0;
          while (pos < mapInfo.tilesize) {
            // Vertical
            ctx.beginPath();
            ctx.moveTo(pos, 0);
            ctx.lineTo(pos, mapInfo.tilesize);
            ctx.stroke();

            // Horizontal
            ctx.beginPath();
            ctx.moveTo(0, pos);
            ctx.lineTo(mapInfo.tilesize, pos);
            ctx.stroke();

            pos += mapInfo.tilesize / lineCount;
          }
          let region = self.CoordToRegion(blockPos);
          const regionName = self.FormatRegionFileName(region);
          ctx.lineWidth = 4;
          ctx.strokeText(regionName, 4, mapInfo.tilesize - 5);
          ctx.fillText(regionName, 4, mapInfo.tilesize - 5);
        } else {
          if (tileLeft % mapInfo.regionsize == 0) {
            // Vertical
            ctx.beginPath();
            ctx.moveTo(0, 0);
            ctx.lineTo(0, mapInfo.tilesize);
            ctx.stroke();
          }
          if (tileBottom % mapInfo.regionsize == 0) {
            // Horizontal
            ctx.beginPath();
            ctx.moveTo(0, mapInfo.tilesize);
            ctx.lineTo(mapInfo.tilesize, mapInfo.tilesize);
            ctx.stroke();
          }
          if (
            tileLeft % mapInfo.regionsize == 0 &&
            tileBottom % mapInfo.regionsize == 0
          ) {
            let region = self.CoordToRegion(blockPos);
            const regionName = self.FormatRegionFileName(region);
            ctx.lineWidth = 4;
            ctx.strokeText(regionName, 4, mapInfo.tilesize - 5);
            ctx.fillText(regionName, 4, mapInfo.tilesize - 5);
          }
        }
        return canvas;
      };

      return regionLayer;
    },

    createCoordinateControl() {
      L.Control.Coordinates = L.Control.extend({
        options: {
          position: "bottomleft",
        },

        onAdd: function(map) {
          var name = "control-coordinates",
            container = L.DomUtil.create("div", name + " webmap-control");

          container.innerHTML =
            "Mouse pos: - E / - N<br/>Last click: - E / - N";
          L.DomEvent.on(container, "mousemove", L.DomEvent.stopPropagation);

          this._map = map;
          this._div = container;

          map.on("mousemove", this._onMouseMove, this);
          map.on("mouseout", this._onMouseOut, this);
          map.on("click", this._onClick, this);

          return container;
        },

        _onMouseMove: function(e) {
          this.lastPos = e.latlng;
          this._updateText();
        },

        _onMouseOut: function() {
          this.lastPos = false;
          this._updateText();
        },

        _onClick: function(e) {
          this.lastClick = e.latlng;
          this._updateText();
        },

        _updateText: function() {
          this._div.innerHTML =
            "Mouse pos: " +
            this._formatCoord(this.lastPos) +
            "<br/>" +
            "Last click: " +
            this._formatCoord(this.lastClick);
        },

        _formatCoord: function(latlng) {
          if (latlng == false) return "- E / - N";
          else
            return (
              "" +
              Math.abs(latlng.lat).toFixed(0) +
              (latlng.lat >= 0 ? " E" : " W") +
              " / " +
              Math.abs(latlng.lng).toFixed(0) +
              (latlng.lng >= 0 ? " N" : " S")
            );
        },

        lastPos: false,
        lastClick: false,
      });
      new L.Control.Coordinates({ position: "topright" }).addTo(this.map);
      return;
    },
    createClusterIcon(kind) {
      return (cluster) => {
        const count = cluster.getChildCount();
        return L.divIcon({
          html: `<div><img src="${this.getClusterIconUrl(
            kind
          )}" alt="" /><span>${count}</span></div>`,
          className: `marker-cluster marker-cluster-${kind}`,
          iconSize: L.point(40, 40),
        });
      };
    },
    getClusterIconUrl(kind) {
      if (kind === "vehicle") {
        return "img/ui_game_symbol_drive.png";
      }
      return "img/ui_game_symbol_character.png";
    },
  },
};
</script>

<style>
#map {
  height: 100vh;
  width: auto;
}

#selection-control {
  pointer-events: auto;
}

.webmap-control {
  color: white;
  font-size: 1.3em;
  text-shadow: 0.07em 0 black, 0 0.07em black, -0.07em 0 black, 0 -0.07em black;
}

.marker-cluster-player div {
  background: rgba(48, 140, 255, 0.85);
  color: #fff;
  border: 2px solid rgba(15, 60, 120, 0.9);
  box-shadow: 0 0 0 2px rgba(15, 60, 120, 0.3);
}

.marker-cluster-vehicle div {
  background: rgba(255, 148, 33, 0.85);
  color: #fff;
  border: 2px solid rgba(140, 70, 10, 0.9);
  box-shadow: 0 0 0 2px rgba(140, 70, 10, 0.3);
}

.marker-cluster-player div,
.marker-cluster-vehicle div {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  text-align: center;
  font-weight: bold;
  position: relative;
}

.marker-cluster-player span,
.marker-cluster-vehicle span {
  position: absolute;
  right: 6px;
  bottom: 2px;
  font-size: 18px;
  line-height: 1;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.8);
}

.marker-cluster-player img,
.marker-cluster-vehicle img {
  width: 22px;
  height: 22px;
  margin-top: 8px;
}

.steam-profile-link {
  font-size: 1.3em;
  text-decoration: none;
}

.steam-profile-link:hover,
.steam-profile-link:focus {
  color: #1e6bd6;
  text-decoration: underline;
}
</style>
