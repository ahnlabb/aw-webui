<template lang="pug">
div.sunburst
  div#sidebar
    input(type="checkbox", id="togglelegend")
    | Legend
    br
    div#legend(style="visibility: hidden")

  div#main
    div#sequence
    div#chart
      div#explanation
        div#base
          | Hover to inspect
        div#hover(style="visibility: hidden")
          div#date
          div#title
          div#time
          div#duration
          div#data(style="text-overflow: ellipsis; white-space: nowrap; overflow: hidden;")
</template>

<style scoped lang="scss">
.sunburst {
  font-family: 'Open Sans', sans-serif;
  font-size: 12px;
  font-weight: 400;
  width: 100%;
  height: 700px;
  margin-top: 10px;

  #main {
    width: 750px;
    margin-right: auto;
    margin-left: auto;
  }

  #sidebar {
    float: right;
    width: 100px;
  }

  #sequence {
    width: 600px;
    height: 70px;
  }

  #legend {
    padding: 10px 0 0 3px;
  }

  #sequence text, #legend text {
    font-weight: 600;
    fill: #fff;
  }

  #chart {
    position: relative;
  }

  #chart path {
    stroke: #fff;
  }

  #explanation {
    position: absolute;
    top: 260px;
    left: 305px;
    width: 140px;
    text-align: center;
    color: #666;
    z-index: 10; // might not be needed

    #base {
        color: #DDD;
        font-size: 2em;
    }

    #hover {
      #date {
        font-size: 0.8em;
      }

      #time {
        font-size: 1em;
      }

      #title {
        font-size: 2em;
        font-weight: bold;
      }

      #duration {
        font-size: 1em;
      }

      #data {
        font-size: 1em;
      }
    }
  }
}
</style>

<script>
// NOTE: This is just a Vue.js component wrapper for timeline-simple.js
//       Code should generally go in the framework-independent file.

// TODO: Sunburst requires a hierarchical data format

import sunburst from './sunburst.js';
import coloring_types from './coloring.js';

let aw_sunburst = {
  name: "aw-sunburst",
  props: ['hierarchy'],
  mounted: function() {
    sunburst.create(this.$el);
  },
  watch: {
    "hierarchy": function() {
      sunburst.create(this.$el);
      sunburst.update(this.$el, this.hierarchy);
    }
  }
}

export default aw_sunburst;
</script>
