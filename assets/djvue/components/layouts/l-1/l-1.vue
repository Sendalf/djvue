<template>
  <v-container>
    <v-layout row wrap>
      <v-flex xs12 pa-1>
        <dj-holder name="Holder" @init="onInitChild"></dj-holder>
      </v-flex>
    </v-layout>
  </v-container>
</template>
<script>
import mixin from "djvue/mixins/core/layout.mixin.js"

export default {
  components: {
    "dj-holder": () => import("djvue/components/core/holder.vue")
  },
  mixins: [mixin],

  methods: {
    onBeforeInit() {
      this._waitList = ["Holder"]
    },

    onChildsInitiated() {
      this.$nextTick(() => {
        this.emit("page-start")
      })
    }
  }

}

</script>
