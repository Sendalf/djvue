<template>
  <div pa-2 mt-2 class="holder" v-bind:class="{producttion:isProductionMode, accepted:isAcceptWidget}">
    <div class="holder-title">
      <h4 v-if="!isProductionMode"> Widget Holder: {{name}}</h4>
    </div>
    <v-layout column wrap>
      <draggable class="list-group" element="div" v-model="widgets" :options="dragOptions" :move="onMove" @start="onStartDrag" @end="onEndDrag">
        <transition-group type="transition" name="holders" tag="div" v-bind:class="{'empty-holder': isEmpty && !isProductionMode}">
          <dj-widget :config="widget" :holder="name" v-for="widget in widgets" :key="widget.id" class="list-group-item" @init="onInitChild"></dj-widget>
        </transition-group>
      </draggable>
    </v-layout>
    <v-layout align-center justify-center row fill-height>
      <v-btn color="primary" v-on:click="insert()" v-if="!isProductionMode">
        <v-icon small class="pr-3">mdi-plus</v-icon>
        Add
      </v-btn>
    </v-layout>
   
  </div>
</template>
<script>
import draggable from "modules/vue-draggable/vuedraggableES6.js";
import djvueMixin from "djvue/mixins/core/djvue.mixin.js"
import listenerMixin from "djvue/mixins/core/listener.mixin.js"
import initiableMixin from "djvue/mixins/core/initiable.mixin.js"
import insertWidgetDialog from "djvue/components/core/dialogs/insertWidgetDialog.vue"

Vue.prototype.$dialog.component('insertWidgetDialog', insertWidgetDialog)

let accepted = null;

let toTree = (object) =>

  _.keys(object).map(key => {
    return {
      name: (_.isObject(object[key])) ? key : `${key}: ${object[key]}`,
      children: (!_.isObject(object[key])) ? undefined : toTree(object[key])
    }
  })


export default {

  mixins: [djvueMixin, listenerMixin, initiableMixin],

  components: {
    "dj-widget": () => import("./widget.vue"),
    draggable
  },

  data() {
    return {
      isAcceptWidget: false
    }
  },

  props: ["name", "type"],

  computed: {

    isEmpty() { return this.widgets.length == 0 },

    dragOptions() {
      return {
        animation: 150,
        group: {
          name: "holders"
        },
        ghostClass: "ghost",
        dragClass: "drag",
        handle: ".handle"
      };
    },

    widgets: {
      get() {
        if (this.type == "skin") return this.app.skin.holders[this.name].widgets
        return this.app.currentPage.holders[this.name].widgets
      },
      set(newValue) {
        this.setHolderContent({
          page: (this.type == "skin") ? null : this.app.currentPage,
          holder: this,
          widgets: newValue
        })
      }
    }



  },


  methods: {

    onBeforeInit() {
      if (this.type == "skin") {
        this._waitList = this.app.skin.holders[this.name].widgets.map(item => item.id)
      } else {
        this._waitList = this.app.currentPage.holders[this.name].widgets.map(item => item.id)
      }

      if (this._waitList.length == 0) {
        this.$emit("init", this.name)
      }

    },

    onChildsInitiated() {
      this.$emit("init", this.name)
    },

    insert() {

      this.$dialog.showAndWait(insertWidgetDialog)
        .then(initialWidgetConfig => {
          if (initialWidgetConfig) {
            initialWidgetConfig.id = this.$djvue.randomName()
            this.widgets.push(this.$djvue.extend({}, initialWidgetConfig))
            // this.setHolderContent({
            //   page: (this.type == "skin") ? null : this.app.currentPage,
            //   holder: this,
            //   widgets: this.widgets
            // })
            this.setNeedSave(true)
          }
        })

    },


    isHoldWidget(widget) {
      return !!_.find(this.widgets, w => w.id == widget.config.id)
    },

    onStartDrag() {
      this.emit("holder-accept", this)
      this.isDragging = true
    },

    onEndDrag() {
      this.emit("holder-accept", null)
      this.isDragging = false
      this.setNeedSave(true)
    },

    onMove({ relatedContext, draggedContext }) {
      this.emit("holder-accept", relatedContext.component.$parent)
      return true
    }

  },

  beforeDestroy() { this.off() },


  created() {


    this.on({
      event: "holder-accept",
      callback: (holder) => { this.isAcceptWidget = holder && (holder.name == this.name) }
    })

    this.on({
      event: "widget-clone",
      callback: (cloned) => {

        // console.log("accept widget-clone")

        let widgetIndex = _.findIndex(this.widgets, w => w.id == cloned.config.id);
        let newWidget = this.$djvue.extend({}, this.widgets[widgetIndex])
        newWidget.id = this.$djvue.randomName();
        newWidget.name += "_clone_" + newWidget.id;

        this.widgets.splice(widgetIndex + 1, 0, newWidget)

      },
      rule: this.isHoldWidget
    })

    this.on({
      event: "widget-delete",
      callback: (deleted) => {
        if( deleted._delete() ){
          let widgetIndex = _.findIndex(this.widgets, w => w.id == deleted.config.id);
          if (widgetIndex > -1) this.widgets.splice(widgetIndex, 1)
          this.setNeedSave(true)  
        }
      },
      rule: this.isHoldWidget
    })




    this.on({
      event: "holder-update-widget-config",
      callback: (emitter, context) => {

        // console.log("accept holder-update-widget-config", context.newConfig)

        let widgetIndex = _.findIndex(this.widgets, w => w.id == context.widget.config.id);
        let newWidgets = JSON.parse(JSON.stringify(this.widgets));
        newWidgets[widgetIndex] = context.newConfig;
        this.widgets = newWidgets;

      },
      rule: this.isHoldWidget
    })

  },

  watch: {

    isDragging(newValue) {
      if (newValue) {
        this.delayedDragging = true;
        return;
      }
      this.$nextTick(() => {
        this.delayedDragging = false;
      });
    }

  }

}

</script>
<style scoped>
.drag {
  opacity: 0;
}

.empty-holder {
  border: 2px dashed #bbb !important;
  min-height: 100px !important;
  background-color: #eee !important;
}

.flip-list-move {
  transition: transform 0.5s;
}

.no-move {
  transition: transform 0s;
}

.ghost {
  opacity: 0 !important;

}

.list-group {
  min-height: 20px;
}

.list-group-item {}

.list-group-item i {}

.container.holder.pa-2 {
  border: 2px solid #bbb !important;
}

.container.holder.pa-2.producttion {
  /* display: none; */
  margin-top: 1em !important;
  border: none !important;
}

.holder-title {
  background: #fafafa;
  margin-top: -1.3em;
  color: #bbb;
  width: fit-content;
  padding: 0em 1em;
}

.container.holder.pa-2.accepted {
  border-color: #00796B !important;
}

.accepted .empty-holder {
  border-color: #00796B !important;
}

.accepted h4 {
  color: #00796B;
}

</style>
