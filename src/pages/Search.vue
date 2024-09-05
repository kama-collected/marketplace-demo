<template>
  <q-page class="row items-stretch">
    <div :class="['col-12 q-pa-md col-md-8']">
      <div class="row q-col-gutter-md items-start">
        <AssetCard
          v-for="(asset, index) of searchResults"
          :key="index"
          :class="['col-12 col-sm-6 col-md-4 col-lg-3 col-xl-3']"
          :asset="asset"
          :reloading="search.searchingAssets"
          :show-distance="!search.searchingAssets && displayAssetDistance"
          @mouseenter.native="animateMarker(asset)"
          @mouseleave.native="animateMarker(asset, false)"
        />
      </div>
      <div class="row justify-center q-mt-md">
        <QPagination
          v-show="searchedAssets.length && search.paginationMeta.nbPages > 1"
          :value="search.searchFilters.page"
          :max="search.paginationMeta.nbPages"
          :max-pages="5"
          color="grey"
          direction-links
          @input="changePage"
        />
      </div>
    </div>
    
    <QPageSticky
      :class="`gt-${mapBreakpoint || 'sm'} col-md-4 full-height`"
      position="top-right"
    >
      <QNoSsr>
        <AppMap
          v-if="mapSized || isSearchMapVisible"
          v-show="isSearchMapVisible"
          class="absolute-full"
          :nav-control="{ show: true, position: 'top-left' }"
          @map-resize="mapResized"
          @map-load="mapLoaded"
          @map-dragstart="mapMoveStarted"
          @map-dragend="mapMoveEnded"
          @map-zoomstart="mapMoveStarted"
          @map-zoomend="mapMoveEnded"
        />

        <!-- Trigger Search on Map Movement -->
        <template v-if="searchAfterMapMoveActive && isSearchMapVisible">
          <QBtn
            v-show="!showRetriggerSearchLabel"
            class="map-search-on-map-move bg-white q-py-sm"
            size="sm"
            @click="toggleSearchAfterMapMove"
          >
            <QCheckbox
              :value="shouldSearchAfterMapMove"
              dense
              @input="toggleSearchAfterMapMove"
            />
            <AppContent class="q-ml-sm" entry="pages" field="search.search_after_map_move" />
          </QBtn>
          <QBtn
            v-show="showRetriggerSearchLabel"
            color="secondary"
            class="map-search-on-map-move q-py-sm"
            size="sm"
            @click="triggerSearchWithMapCenter"
          >
            <AppContent v-show="showRetriggerSearchLabel" entry="pages" field="search.redo_search" />
          </QBtn>
        </template>
      </QNoSsr>
    </QPageSticky>
  </q-page>
</template>

<script>
import Vue from 'vue'
import { mapState, mapGetters } from 'vuex'
import * as mutationTypes from 'src/store/mutation-types'
import { fill, get, set } from 'lodash'
import getDistance from 'geolib/es/getDistance'
import p from 'src/utils/promise'
import AssetCard from 'src/components/AssetCard'
import PageComponentMixin from 'src/mixins/pageComponent'

export default {
  components: {
    AppMap: () => import(/* webpackChunkName: 'mapbox' */ 'mapbox-gl')
      .then(mapbox => {
        if (window && !window.mapboxgl) window.mapboxgl = mapbox.default
        return import(/* webpackChunkName: 'mapbox' */ 'src/components/AppMap')
      }),
  },
  mixins: [PageComponentMixin],
  data() {
    return {
      mapBreakpoint: 'sm', 
      mapSized: false,
      populatedMapMarkers: {},
      shouldSearchAfterMapMove: true,
      mapCenterChanged: false,
      mapMovingProgrammatically: false,
      mapCenterGps: {
        latitude: 0,
        longitude: 0,
      },
      mapMaxDistance: null,
      mapFitBoundsActive: true,
      mapFitBoundsTimeout: null,
    };
  },
  computed: {
    ...mapState(['search', 'common']),
    ...mapGetters(['getBaseImageUrl', 'searchedAssets', 'isSearchMapVisible', 'defaultSearchMode', 'searchAfterMapMoveActive', 'displayAssetDistance']),
    
    showRetriggerSearchLabel() {
      return this.shouldSearchAfterMapMove ? false : this.mapCenterChanged;
    },

    searchResults() {
      return this.search.searchingAssets && !this.searchedAssets.length
        ? fill(Array(this.search.searchFilters.nbResultsPerPage / 2), null)
        : this.searchedAssets;
    },
  },
  watch: {
    'search.assets'() {
      this.refreshMap();
    },
    isSearchMapVisible(visible) {
      if (!visible) {
        this.destroyMarkers();
        this.map = null;
      }
    }
  },
  async mounted() {
    await Promise.all([
      this.$store.dispatch('fetchConfig'),
      this.$store.dispatch('fetchAssetsRelatedResources'),
    ]);

    if (!this.$store.state.search.searchMode) {
      await this.$store.dispatch('selectSearchMode', { searchMode: this.$store.getters.defaultSearchMode });
    }

    if (window.__PRERENDER_INJECTED) document.dispatchEvent(new Event('prerender-ready'));
    this.$store.dispatch('getHighestPrice');
    'requestIdleCallback' in window ? requestIdleCallback(this.showMapOnLargeScreen) : setTimeout(this.showMapOnLargeScreen, 0);
  },
  updated() {
    this.refreshMap();
  },
  beforeDestroy() {
    this.$store.commit(mutationTypes.TOGGLE_FILTER_DIALOG, { visible: false });
    this.destroyMarkers();
    delete window.stlMapMarkers;
    clearTimeout(this.mapFitBoundsTimeout);
  },
  methods: {
    async afterAuth() {
      if (!this.search.searchMode) {
        await this.$store.dispatch('selectSearchMode', { searchMode: this.defaultSearchMode });
      }
      await this.searchAssets();
    },
    mapResized() {
      this.mapSized = true;
    },
    showMapOnLargeScreen() {
      if (this.$q.screen.gt[this.mapBreakpoint]) {
        this.$store.commit(mutationTypes.TOGGLE_SEARCH_MAP, { visible: true });
      }
    },
    async changePage(page) {
      this.$store.commit({
        type: mutationTypes.SEARCH__SET_SEARCH_FILTERS,
        page
      });
      await this.searchAssets({ resetPagination: false });
    },
    async searchAssets({ resetPagination } = {}) {
      await this.$store.dispatch('searchAssets', { resetPagination });
    },
    getMapFeatures() {
      return this.searchedAssets.filter(a => a.locations.length).map(a => ({
        type: 'Feature',
        geometry: {
          type: 'Point',
          coordinates: [a.locations[0].longitude, a.locations[0].latitude]
        },
        properties: {
          assetId: a.id
        }
      }));
    },
    mapLoaded(map) {
      this.map = map;
      this.refreshMap();
    },
    refreshMap() {
      if (!this.map || !this.isSearchMapVisible) return;

      const mapFeatures = this.getMapFeatures();
      this.destroyMarkers({ keep: this.searchedAssets.map(a => a.id) });

      if (!mapFeatures.length) {
        this.mapFitBoundsActive = true;
        return;
      }

      if (this.mapFitBoundsActive) {
        this.mapMovingProgrammatically = true;

        const firstCoordinates = mapFeatures[0].geometry.coordinates;
        const bounds = mapFeatures.reduce((bounds, f) => bounds.extend(f.geometry.coordinates), new mapboxgl.LngLatBounds(firstCoordinates, firstCoordinates));

        const fitBoundsDuration = 2000;
        this.map.fitBounds(bounds, {
          padding: { top: 75, right: 25, bottom: 150, left: 25 },
          duration: fitBoundsDuration
        });
        this.mapFitBoundsTimeout = setTimeout(() => {
          this.mapMovingProgrammatically = false;
          this.mapFitBoundsActive = true;
        }, fitBoundsDuration);
      } else {
        this.mapFitBoundsActive = true;
      }

      p.map(mapFeatures, async f => {
        const assetId = f.properties.assetId;
        const asset = this.searchedAssets.find(a => a.id === assetId);
        const markerId = `marker-${assetId}`;
        const imgSrc = this.getBaseImageUrl(asset);

        if (!imgSrc || !this.map || !this.$refs || !this.$refs[markerId]) return;

        const element = this.$refs[markerId];
        const marker = new window.mapboxgl.Marker(element)
          .setLngLat(f.geometry.coordinates)
          .addTo(this.map);

        if (!this.populatedMapMarkers[assetId]) this.populatedMapMarkers[assetId] = {};
        this.populatedMapMarkers[assetId].marker = marker;

        marker.getElement().addEventListener('mouseenter', () => this.animateMarker(asset));
        marker.getElement().addEventListener('mouseleave', () => this.animateMarker(asset, false));
      });
    },
    destroyMarkers({ keep = [] } = {}) {
      Object.keys(this.populatedMapMarkers).forEach(key => {
        if (!keep.includes(key)) {
          const { marker } = this.populatedMapMarkers[key];
          if (marker) marker.remove();
          delete this.populatedMapMarkers[key];
        }
      });
    },
    toggleSearchAfterMapMove() {
      this.shouldSearchAfterMapMove = !this.shouldSearchAfterMapMove;
    },
    mapMoveStarted() {
      if (!this.mapMovingProgrammatically) {
        this.mapFitBoundsActive = false;
      }
    },
    mapMoveEnded() {
      if (!this.mapMovingProgrammatically) {
        this.mapCenterChanged = true;
        if (this.shouldSearchAfterMapMove) {
          this.triggerSearchWithMapCenter();
        }
      }
    },
    triggerSearchWithMapCenter() {
      const center = this.map.getCenter();
      if (!center) return;
      this.mapCenterGps.latitude = center.lat;
      this.mapCenterGps.longitude = center.lng;
      this.searchAssets();
      this.mapCenterChanged = false;
    }
  }
};
</script>
