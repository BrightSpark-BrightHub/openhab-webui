<template>
  <knob-control v-bind="config" :value="value" @input="onChange" />
</template>

<script>
import mixin from '../widget-mixin'

import KnobControl from 'vue-knob-control'

export default {
  mixins: [mixin],
  components: {
    KnobControl
  },
  data () {
    return {
      pendingCommand: null,
      delayCommand: false,
      delayUpdate: null
    }
  },
  mounted () {
    delete this.config.value
  },
  computed: {
    value () {
      if (this.delayUpdate && this.pendingCommand) return this.pendingCommand // to keep the control reactive when operating
      const value = this.context.store[this.config.item].state
      // use as a brightness control for HSB values
      if (value.split && value.split(',').length === 3) return parseFloat(value.split(',')[2])
      return parseFloat(value)
    }
  },
  methods: {
    onChange (value) {
      if (value === this.value) return
      this.pendingCommand = value
      if (!this.delayCommand) {
        this.delayCommand = true
        this.setUpdateDelayTimout()
        this.$store.dispatch('sendCommand', { itemName: this.config.item, cmd: value.toString() })
        setTimeout(() => {
          this.delayCommand = false
          if (this.delayUpdate) clearTimeout(this.delayUpdate)
          if (this.pendingCommand) {
            this.setUpdateDelayTimout()
            this.$store.dispatch('sendCommand', { itemName: this.config.item, cmd: this.pendingCommand.toString() })
          }
        }, 200)
      }
    },
    setUpdateDelayTimout () {
      if (this.delayUpdate) clearTimeout(this.delayUpdate)
      this.delayUpdate = setTimeout(() => {
        console.debug('End of update delay')
        this.delayUpdate = null
        this.pendingCommand = null
      }, 2000)
    }
  }
}
</script>
