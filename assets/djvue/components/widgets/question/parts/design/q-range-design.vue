<template>
  <div>
    <div v-if="!options" class="text-xs-center">
      <v-progress-circular indeterminate color="secondary ligten-2"></v-progress-circular>
    </div>
    <v-tabs v-else v-model="active" color="transparent">
      <v-tab key="general" ripple>General</v-tab>
      <v-tab key="rate" ripple>Range</v-tab>
      <v-tab key="statistic" ripple>Statistic</v-tab>
      <v-tab-item key="general" ripple>
        <v-card flat>
          <v-container>
            <h3 class="headline">Range</h3>
            <v-switch :hint="(options.required) ? 'All constraints will be checked' : 'Response validation will be disabled'" persistent-hint label="Required" v-model="options.required" @change="$emit('update:options',options)"></v-switch>
            <v-divider></v-divider>
            <v-textarea v-model="options.title" label="Question"></v-textarea>
            <v-textarea v-model="options.note" label="Note"></v-textarea>
          </v-container>
        </v-card>
      </v-tab-item>
      <v-tab-item key="rate" ripple>
        <v-card flat>
          <v-container>
            <v-layout row>
              <v-flex xs6>
                Options
                <v-container>
                  <v-input label="Min" persistent-hint hint="Min value">
                    <div style="padding: 0 0 0 1em;">
                      <input type="number" style="border: 1px solid #e0e0e0;" v-model="options.range[0]" :max="options.range[1]" @input="$emit('update:options', options)"></input>
                    </div>
                  </v-input>
                  <v-divider></v-divider>
                  <v-input class="mt-2" label="Max" persistent-hint hint="Max value">
                    <div style="padding: 0 0 0 1em;">
                      <input type="number" style="border: 1px solid #e0e0e0;" v-model="options.range[1]" :min="options.range[0]" @input="$emit('update:options', options)"></input>
                    </div>
                  </v-input>
                  <v-divider></v-divider>
                  <v-input class="mt-2" label="Step" persistent-hint hint="step value">
                    <div style="padding: 0 0 0 1em;">
                      <input type="number" style="border: 1px solid #e0e0e0;" v-model="options.step" :min="0" :max="options.range[1]-options.range[0]" @input="$emit('update:options', options)"></input>
                    </div>
                  </v-input>
                  <v-divider></v-divider>
                </v-container>
              </v-flex>
              <v-flex xs6>
                <v-container>
                  Preview
                  <v-layout column v-if="options" class="mt-4">
                    <v-range-slider v-model="range" :max="options.range[1]" :min="options.range[0]" :step="options.step" thumb-size="36" thumb-label="always" thumb-color="accent" ticks="always" tick-size="2"></v-range-slider>
                  </v-layout>
                </v-container>
              </v-flex>
            </v-layout>
          </v-container>
        </v-card>
      </v-tab-item>
      <v-tab-item key="statistic" ripple>
        <v-card flat>
          <v-container>
            <v-switch :hint="(options.showResponsesStat) ? 'Show statistical distribution in production mode' : ''" persistent-hint label="Statistical hints" v-model="options.showResponsesStat" @change="$emit('update:options',options)"></v-switch>
            <v-divider></v-divider>
            <echart :options="statOptions" :height="height"></echart>
          </v-container>
        </v-card>
      </v-tab-item>
    </v-tabs>
    <!-- <pre>
	    	{{JSON.stringify(stat,null,"\t")}}
	    </pre>	 -->
  </div>
</template>
<script>
import djvueMixin from "djvue/mixins/core/djvue.mixin.js";
import listenerMixin from "djvue/mixins/core/listener.mixin.js";
import statMixin from "../mixins/statistic.mixin.js"

export default {

  mixins: [djvueMixin, listenerMixin, statMixin],

  props: ["config", "options", "stat"],


  methods: {


     calculateStat() {
      let s = this.stat.responses.filter(a => a)


      let stats = [];

      s.forEach(v => {
        stats = stats.concat(v)
      })

      let left = stats.map(item => item[0])
      let right = stats.map(item => item[1])

      let r = _.union(_.uniq(left),_.uniq(right)).sort((a,b) => a-b).map(item=>({title:item}))

      let result = r.map((n, index) => {
        let l = _.countBy(left)[n.title]
        let r = _.countBy(right)[n.title]

        return {
          title: n.title,
          left: ((l) ? l : 0),
          right: ((r) ? r : 0)
        }
      })
      let statOptions = {
        
      	legend: {
        	data:['Left boundary', 'Right boundary']
    	},
        grid: {
          left: '3%',
          right: '4%',
          bottom: '3%',
          containLabel: true
        },
        xAxis: {
          type: 'value',
        },
        yAxis: {
          type: 'value'
        },
        series: []
      }

      statOptions.series.push({
        name: 'Left boundary',
        type: 'line',
        step: 'middle',
        areaStyle:{
          opacity:0.25
        },
        data: result.map(item => [item.title, item.left]),
        markLine: {
          data: [{
            xAxis: (_.sum(left) / left.length)
          }]
        }
      })
      statOptions.series.push({
        name: 'Right boundary',
        type: 'line',
        step: 'middle',
        areaStyle:{
          opacity:0.25
        },
        data: result.map(item => [item.title, item.right]),
        markLine: {
          data: [{
            xAxis: (_.sum(right) / right.length)
          }]
        }
      })

      this.height = 250;
      return statOptions
    },




    onChange() {
      this.$emit("update:options", this.options)
    }
  },

  data: () => ({
    active: null,
    height: null,
    range: [0, 1]
  }),

  watch: {
    options: {
      handler(value) {
        if (value && value.range) {
          this.range = [value.range[0], value.range[0]];
        }
      },
      deep: true
    }
  },

  mounted() {
    if (this.options && this.options.range) {
      this.range = [this.options.range[0], this.options.range[0]];
    }
    this.$emit("init")
  }
}

</script>
