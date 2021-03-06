<template>
  <f7-page @page:afterin="onPageAfterIn" @page:beforeout="onPageBeforeOut" class="map-editor">
    <f7-navbar :title="(!ready) ? '' : (createMode) ? 'Create map page' : page.config.label" back-link="Back" no-hairline>
      <f7-nav-right>
        <f7-link @click="save()" v-if="$theme.md" icon-md="material:save" icon-only></f7-link>
        <f7-link @click="save()" v-if="!$theme.md">Save<span v-if="$device.desktop">&nbsp;(Ctrl-S)</span></f7-link>
      </f7-nav-right>
    </f7-navbar>
    <f7-toolbar tabbar position="top">
      <f7-link @click="currentTab = 'design'; fromYaml()" :tab-link-active="currentTab === 'design'" class="tab-link">Design</f7-link>
      <f7-link @click="currentTab = 'code'; toYaml()" :tab-link-active="currentTab === 'code'" class="tab-link">Code</f7-link>
    </f7-toolbar>
    <f7-toolbar bottom class="toolbar-details" v-show="currentTab === 'design'">
      <div style="margin-left: auto">
        <f7-toggle :checked="previewMode" @toggle:change="(value) => previewMode = value"></f7-toggle> Run mode<span v-if="$device.desktop">&nbsp;(Ctrl-R)</span>
      </div>
    </f7-toolbar>

    <f7-tabs class="map-editor-tabs">
      <f7-tab id="design" class="map-editor-design-tab" @tab:show="() => this.currentTab = 'design'" :tab-active="currentTab === 'design'">
        <f7-block v-if="!ready" class="text-align-center">
          <f7-preloader></f7-preloader>
          <div>Loading...</div>
        </f7-block>
        <f7-block class="block-narrow" v-if="ready && !previewMode">
          <page-settings :page="page" :createMode="createMode" />
        </f7-block>

        <f7-block class="block-narrow" style="padding-bottom: 8rem" v-if="ready && !previewMode">
          <f7-col>
            <f7-block-title>Markers</f7-block-title>
            <f7-menu v-if="clipboardType === 'oh-map-marker'">
              <f7-menu-item style="margin-left: auto" icon-f7="map" dropdown>
                <f7-menu-dropdown right>
                  <f7-menu-dropdown-item @click="pasteWidget(page, null)" href="#" text="Paste"></f7-menu-dropdown-item>
                </f7-menu-dropdown>
              </f7-menu-item>
            </f7-menu>

            <f7-list media-list>
              <f7-list-item media-item v-for="(marker, idx) in page.slots.default" :key="idx"
                :title="marker.config.label" :subtitle="marker.config.item || marker.config.location">
                <oh-icon v-if="marker.config.icon && marker.config.icon.indexOf('oh:') === 0" slot="media" :icon="marker.config.icon.substring(3)" height="32" width="32" />
                <f7-icon v-else slot="media" :f7="markerDefaultIcon(marker)" :size="32" />
                <f7-menu slot="content-start">
                  <f7-menu-item icon-f7="list_bullet" dropdown>
                    <f7-menu-dropdown>
                      <f7-menu-dropdown-item @click="configureWidget(marker,  { component: page })" href="#" text="Configure marker"></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item @click="editWidgetCode(marker, { component: page })" href="#" text="Edit YAML"></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item divider></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item @click="cutWidget(marker, { component: page })" href="#" text="Cut"></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item @click="copyWidget(marker, { component: page })" href="#" text="Copy"></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item divider></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item @click="moveWidgetUp(marker, { component: page })" href="#" text="Move Up"></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item @click="moveWidgetDown(marker, { component: page })" href="#" text="Move Down"></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item divider></f7-menu-dropdown-item>
                      <f7-menu-dropdown-item @click="removeWidget(marker, { component: page })" href="#" text="Remove marker"></f7-menu-dropdown-item>
                    </f7-menu-dropdown>
                  </f7-menu-item>
                </f7-menu>
              </f7-list-item>
              <f7-list-button color="blue" title="Add marker" @click="addWidget(page, 'oh-map-marker')" />
              <f7-list-button color="blue" title="Add circle marker" @click="addWidget(page, 'oh-map-circle-marker')" />
            </f7-list>
          </f7-col>
        </f7-block>

        <oh-map-page class="map-page" v-else-if="ready && previewMode" :context="context" :key="pageKey" />
      </f7-tab>

      <f7-tab id="code" @tab:show="() => { this.currentTab = 'code' }" :tab-active="currentTab === 'code'">
        <editor v-if="currentTab === 'code'" class="page-code-editor" mode="text/x-yaml" :value="pageYaml" @input="(value) => pageYaml = value" />
        <pre class="yaml-message padding-horizontal" :class="[yamlError === 'OK' ? 'text-color-green' : 'text-color-red']">{{yamlError}}</pre>
      </f7-tab>
    </f7-tabs>

    <widget-config-popup :opened="widgetConfigOpened" :component="currentComponent" :widget="currentWidget" @closed="widgetConfigClosed" @update="updateWidgetConfig" />
    <widget-code-popup :opened="widgetCodeOpened" :component="currentComponent" :widget-yaml="widgetYaml" @closed="widgetCodeClosed" @update="updateWidgetCode" />
  </f7-page>
</template>

<style lang="stylus">
.page-code-editor.vue-codemirror
  display block
  top calc(var(--f7-navbar-height) + var(--f7-tabbar-height))
  height calc(80% - 2*var(--f7-navbar-height))
  width 100%
.yaml-message
  display block
  position absolute
  top 80%
  white-space pre-wrap
.map-editor
  .oh-map-page-lmap
    top calc(var(--f7-navbar-height) + var(--f7-toolbar-height)) !important
    height calc(100% - var(--f7-navbar-height) - 2 * var(--f7-toolbar-height)) !important
</style>

<script>
import PageDesigner from '../pagedesigner-mixin'

import YAML from 'yaml'

import OhMapMarker from '@/components/widgets/map/oh-map-marker.vue'
import OhMapCircleMarker from '@/components/widgets/map/oh-map-circle-marker.vue'

const ConfigurableWidgets = {
  OhMapMarker,
  OhMapCircleMarker
}

import PageSettings from '@/components/pagedesigner/page-settings.vue'
import WidgetConfigPopup from '@/components/pagedesigner/widget-config-popup.vue'
import WidgetCodePopup from '@/components/pagedesigner/widget-code-popup.vue'

export default {
  mixins: [PageDesigner],
  components: {
    'editor': () => import('@/components/config/controls/script-editor.vue'),
    'oh-map-page': () => import('@/components/widgets/map/oh-map-page.vue'),
    PageSettings,
    WidgetConfigPopup,
    WidgetCodePopup
  },
  props: ['createMode', 'uid'],
  data () {
    return {
      forceEditMode: true,
      page: {
        uid: 'page_' + this.$f7.utils.id(),
        component: 'oh-map-page',
        config: {},
        slots: { default: [] }
      }
    }
  },
  methods: {
    markerDefaultIcon (marker) {
      const widgetDefinition = Object.values(ConfigurableWidgets).find((c) => c.widget.name === marker.component)
      if (widgetDefinition) {
        return widgetDefinition.widget.icon
      }
      return null
    },
    addWidget (component, widgetType, parentContext, slot) {
      if (!slot) slot = 'default'
      if (!component.slots) component.slots = {}
      if (!component.slots[slot]) component.slots[slot] = []
      if (widgetType) {
        component.slots[slot].push({
          component: widgetType,
          config: {},
          slots: { default: [] }
        })
        this.forceUpdate()
      }
    },
    getWidgetDefinition (componentType) {
      const component = Object.values(ConfigurableWidgets).find((w) => w.widget && w.widget.name === componentType)
      if (!component) return null
      return component.widget
    },
    toYaml () {
      this.pageYaml = YAML.stringify({
        markers: this.page.slots.default
      })
    },
    fromYaml () {
      try {
        const updatedMarkers = YAML.parse(this.pageYaml)
        this.$set(this.page.slots, 'default', updatedMarkers.markers)
        this.forceUpdate()
        return true
      } catch (e) {
        this.$f7.dialog.alert(e).open()
        return false
      }
    }
  }
}
</script>
