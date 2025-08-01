<template>
  <div
    class="content-container"
    @mouseover="mouseHovered = true"
    @mouseleave="mouseHovered = false"
  >
    <ContentBar
      class="toolbar"
      :entry="entry"
      ref="contentBar"
      @chooser-changed="onResize"
      @scaffold-view-clicked="scaffoldViewClicked"
      @vue:mounted="setPanesBoundary"
    />
  <!--
    <DatasetHeader
      v-if="entry.datasetTitle"
      class="dataset-header"
      :entry="entry"
    ></DatasetHeader>
  -->
    <div class="component-container" ref="container">
      <Suspense>
        <Component
          :is="viewerType"
          :entry="entry"
          :mouseHovered="mouseHovered"
          :visible="visible"
          :lazy="true"
          ref="viewer"
          @flatmap-provenance-ready="flatmapProvenanceReady"
          @resource-selected="resourceSelected"
          @species-changed="speciesChanged"
        />
      </Suspense>
    </div>
  </div>
</template>

<script>
/* eslint-disable no-alert, no-console */
import { defineAsyncComponent } from 'vue'
import { ElButton as Button } from "element-plus";
import ContentBar from "./ContentBar.vue";
import { mapStores } from 'pinia';
import { useEntriesStore } from '../stores/entries';
import { useSplitFlowStore } from '../stores/splitFlow';

const Flatmap = defineAsyncComponent(() => import("./viewers/Flatmap.vue"));
const Iframe = defineAsyncComponent(() => import("./viewers/Iframe.vue"));
const MultiFlatmap = defineAsyncComponent(() => import("./viewers/MultiFlatmap.vue"));
const Plot = defineAsyncComponent(() => import("./viewers/Plot.vue"));
const Scaffold = defineAsyncComponent(() => import("./viewers/Scaffold.vue"));
const Simulation = defineAsyncComponent(() => import("./viewers/Simulation.vue"));

export default {
  name: "ContentVuer",
  props: {
    /**
     * Object containing information for
     * the required viewing.
     */
    entry: Object,
    visible: {
      type: Boolean,
      default: true,
    },
  },
  components: {
    //DatasetHeader,
    Button,
    ContentBar,
    Flatmap,
    Iframe,
    MultiFlatmap,
    Plot,
    Scaffold,
    Simulation,
  },
  methods: {
    flatmapProvenanceReady: function(prov) {
      this.$refs.contentBar?.setupFlatmapContextCard(prov);
      this.entriesStore.updateMapForEntry(this.entry, prov);
    },
    getId: function () {
      return this.entry.id;
    },
    getState: function () {
      return this.$refs.viewer?.getState();
    },
    resourceSelected: function (payload) {
      this.$emit("resource-selected", payload);
    },
    scaffoldViewClicked: function (viewUrl) {
      if (this.entry.viewUrl !== viewUrl) {
        this.entriesStore.updateViewForEntry({id: this.entry.id, viewUrl});
      } else {
        //Manually set it as it cannot be set with reactivity
        this.$refs.viewer?.updateWithViewUrl(viewUrl);
      }
    },
    /**
     * Perform a local search on this contentvuer
     * @param {string} term - The search term to look for
     */
    search: function (term) {
      return this.$refs.viewer?.search(term);
    },
    /**
     * Push the suggested terms into the suggestions array
     */
    searchSuggestions: function(term, suggestions) {
      this.$refs.viewer?.searchSuggestions(term, suggestions);
    },
    setPanesBoundary: function() {
      this.$refs.contentBar?.setBoundary(this.$refs["container"]);
    },
    speciesChanged: function (species) {
      this.activeSpecies = species;
      this.$emit("species-changed", species);
    },
    /**
     * Check if this viewer is currently visible
     */
    isVisible: function() {
      const paneName = this.splitFlowStore.getPaneNameById(this.entry.id);
      let visible = false;
      if (paneName !== undefined) {
        visible = this.splitFlowStore.isPaneActive(paneName);
      }
      return visible;
    },
    onResize: function () {
      this.$refs.viewer?.onResize();
    },
    //The following handle EventBus.on from SplitDialog
    onConnectivityItemClose: function() {
      this.$refs.viewer?.onConnectivityItemClose();
    },
    onConnectivitySourceChange: function(payload) {
      this.$refs.viewer?.changeConnectivitySource(payload);
    },
    onFlatmapMarkerUpdate: function() {
      this.$refs.viewer?.flatmapMarkerUpdate();
    },
    onGlobalViewerSettingsUpdate: function() {
      this.$refs.viewer?.updateViewerSettings();
    },
    onHoverUpdate: function(payload) {
      this.$refs.viewer?.sidebarHoverHighlight(payload);
    },
    onShowConnectivity: function(payload) {
      this.$refs.viewer?.showConnectivity(payload);
    },
    onShowConnectivityTooltips: function(payload) {
      this.$refs.viewer?.showConnectivityTooltips(payload);
    },
    onShowReferenceConnectivity: function(payload) {
      this.$refs.viewer?.showConnectivitiesByReference(payload);
    },
    onSidebarAnnotationClose: function() {
      this.$refs.viewer?.onSidebarAnnotationClose();
    },
    onStartHelp: function() {
      this.$refs.viewer?.startHelp();
    },
    onFilterVisibility: function(payload) {
      this.$refs.viewer?.setVisibilityFilter(payload);
    },
    onLoadConnectivityDetail: function(payload) {
      this.$refs.viewer?.getKnowledgeTooltip(payload);
    },
  },
  data: function () {
    return {
      mouseHovered: false,
      activeSpecies: "Rat",
    };
  },
  computed: {
    ...mapStores(useEntriesStore, useSplitFlowStore),
    viewerType() {
      switch (this.entry.type) {
        case "Biolucida":
        case "Iframe":
        case "Segmentation":
          return 'Iframe';
        default:
          return this.entry.type;
      }
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
.toolbar {
  width: 100%;
  background-color: #f5f7fa !important;
  position: absolute;
  transition: all 1s ease;
  height: 32px;
  border-bottom: 1px solid rgb(220, 223, 230);
  z-index: 7;
  box-sizing: border-box;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
}

.component-container {
  height: calc(100% - 32px);
  width: 100%;
  bottom: 0px;
  position: absolute;
  z-index:6;
  overflow: hidden;
}

.dataset-header {
  height: 23px;
}

.content-container {
  height: 100%;
  width: 100%;
}

.open-scaffold {
  position: absolute;
  left: calc(50% - 64px);
  z-index: 2;
  top: 8px;
  font-size: 16px;
  padding-top: 9px;
  padding-bottom: 9px;
  &.el-button--primary.is-plain {
    &:hover,
    &:active,
    &:focus {
      color: #8300bf;
      background: #f3e6f9;
      border-color: #cd99e5;
    }
    &:hover {
      box-shadow: -3px 2px 4px rgba(0, 0, 0, 0.2);
    }
  }
}
</style>
