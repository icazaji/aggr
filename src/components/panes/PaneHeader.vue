<template>
  <div class="pane-header hide-scrollbar pane-overlay d-flex">
    <span class="pane-header__name mrauto" @dblclick="renamePane">
      <slot name="title">
        {{ name }}
      </slot>
    </span>
    <div class="toolbar flex-grow-1" @dblclick="maximizePane">
      <slot />
      <button
        v-if="showSearch"
        type="button"
        @click="openSearch"
        class="btn toolbar__label toolbar__label--768 -text"
      >
        <i class="icon-search"></i>
      </button>
      <btn
        v-if="settings"
        type="button"
        @click="openSettings"
        class="toolbar__label toolbar__label--768 -text"
        :loading="isLoading"
      >
        <i class="icon-cog"></i>
      </btn>
      <button type="button" @click="toggleDropdown" class="btn toolbar__label">
        <i class="icon-more"></i>
      </button>

      <dropdown v-model="paneDropdownTrigger">
        <div class="d-flex btn-group">
          <button
            type="button"
            class="btn -green"
            @click.stop="changeZoom($event, -1)"
          >
            <i class="icon-minus"></i>
          </button>
          <button
            type="button"
            class="btn -green text-monospace flex-grow-1 text-center"
            @click.stop="resetZoom"
          >
            × {{ zoom.toFixed(2) }}
          </button>
          <button
            type="button"
            class="btn -green"
            @click.stop="changeZoom($event, 1)"
          >
            <i class="icon-plus"></i>
          </button>
        </div>
        <button
          v-if="settings !== null"
          type="button"
          class="dropdown-item"
          @click="openSettings"
        >
          <i class="icon-cog"></i>
          <span>Settings</span>
        </button>
        <button
          v-if="showSearch"
          type="button"
          class="dropdown-item"
          @click="$store.dispatch('app/showSearch', paneId)"
        >
          <i class="icon-search"></i>
          <span>Sources</span>
        </button>
        <div
          v-if="$slots.menu"
          class="dropdown-divider"
          :data-label="`${paneId} options`"
        ></div>
        <slot name="menu"></slot>
        <div class="dropdown-divider" data-label="pane options"></div>
        <button type="button" class="dropdown-item" @click="maximizePane">
          <i class="icon-enlarge"></i>
          <span>Maximize</span>
        </button>
        <button type="button" class="dropdown-item" @click="duplicatePane">
          <i class="icon-copy-paste"></i>
          <span>Duplicate</span>
        </button>
        <button type="button" class="dropdown-item" @click="downloadPane">
          <i class="icon-download"></i>
          <span>Download</span>
        </button>
        <div class="dropdown-divider"></div>
        <button type="button" class="dropdown-item" @click="removePane">
          <i class="icon-trash"></i>
          <span>Remove</span>
        </button>
      </dropdown>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import Component from 'vue-class-component'

import Btn from '@/components/framework/Btn.vue'
import { downloadAnything, getSiblings, slugify } from '@/utils/helpers'
import dialogService from '@/services/dialogService'

@Component({
  name: 'PaneHeader',
  props: {
    paneId: {
      type: String
    },
    settings: {
      type: Function,
      default: null
    },
    showSearch: {
      type: Boolean,
      default: true
    }
  },
  components: {
    Btn
  }
})
export default class extends Vue {
  private settings?: () => Promise<any>
  paneId: string
  paneDropdownTrigger = null
  isLoading = false

  get zoom() {
    return this.$store.state.panes.panes[this.paneId].zoom || 1
  }

  get type() {
    return this.$store.state.panes.panes[this.paneId].type
  }

  get name() {
    const name = this.$store.state.panes.panes[this.paneId].name
    const market =
      this.$store.state.panes.marketsListeners[
        this.$store.state.panes.panes[this.paneId].markets[0]
      ]

    if (name) {
      return name
    } else if (market) {
      return market.local
    } else {
      return this.type
    }
  }

  openSearch() {
    this.$store.dispatch('app/showSearch', this.paneId)
  }

  changeZoom(event, direction) {
    const zoom = this.zoom + (event.shiftKey ? 0.0625 : 0.125) * direction
    this.$store.dispatch('panes/setZoom', { id: this.paneId, zoom: zoom })

    this.$emit('zoom', zoom)
  }

  resetZoom() {
    this.$store.dispatch('panes/setZoom', { id: this.paneId, zoom: 1 })

    this.$emit('zoom', 1)
  }

  async removePane() {
    if (await dialogService.confirm(`Delete pane "${this.paneId}" ?`)) {
      this.$store.dispatch('panes/removePane', this.paneId)
    }
  }

  duplicatePane() {
    this.$store.dispatch('panes/duplicatePane', this.paneId)
  }

  highlightPane(value: boolean) {
    this.$el.parentElement.parentElement.classList[value ? 'add' : 'remove'](
      '-highlight'
    )
  }

  maximizePane(event) {
    if (event.type === 'dblclick' && event.currentTarget !== event.target) {
      return
    }

    const el = this.$el.parentElement.parentElement
    const isMaximized = el.classList.toggle('-maximized')

    const siblings = getSiblings(el)

    for (const sibling of siblings) {
      if (!sibling.getAttribute('type')) {
        continue
      }
      sibling.classList.remove('-maximized')
      sibling.style.display = isMaximized ? 'none' : 'block'
    }

    const cls = Event as any

    window.dispatchEvent(new cls('resize'))
  }

  async renamePane() {
    const name = await dialogService.prompt({
      action: 'Rename',
      input: this.name
    })

    if (name !== this.name) {
      this.$store.commit('panes/SET_PANE_NAME', { id: this.paneId, name: name })
    }
  }

  async downloadPane() {
    downloadAnything(
      {
        name: `${this.type}:${this.paneId}`,
        type: this.type,
        data: this.$store.state[this.paneId],
        createdAt: Date.now(),
        updatedAt: null
      },
      slugify(this.paneId)
    )
  }

  toggleDropdown(event) {
    if (this.paneDropdownTrigger) {
      this.paneDropdownTrigger = null
    } else {
      this.paneDropdownTrigger = event.currentTarget
    }
  }

  async openSettings() {
    if (!this.settings) {
      return
    }

    this.isLoading = true
    dialogService.open((await this.settings()).default, {
      paneId: this.paneId
    })
    this.isLoading = false
  }
}
</script>
