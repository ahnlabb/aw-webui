<template lang="pug">
div
  h2 Window Activity for {{ dateShort }}

  p Host: {{ host }}

  b-alert(variant="danger" :show="errormsg.length > 0")
    | {{ errormsg }}

  b-button-group
    b-button(:to="'/activity/' + host + '/' + previousDay()", variant="outline-dark")
      icon(name="arrow-left")
      |  Previous day
    b-button(:to="'/activity/' + host + '/' + nextDay()", :disabled="nextDay() > today", variant="outline-dark")
      |  Next day
      icon(name="arrow-right")
  b-button(v-on:click="refresh()", style="margin-left: 1rem;", variant="outline-dark")
    icon(name="refresh")
    |  Refresh

  hr


  div.row
    div.col-md-6
      h5 Summary
      | Total active time: {{ readableDuration }}
      br
      | Events queried: {{ eventcount }}
    div.col-md-6
      b Options
      div
        label.custom-control.custom-checkbox
          input.custom-control-input(type="checkbox", v-model="filterAFK")
          span.custom-control-indicator
          span.custom-control-description
            | Filter away AFK time

  hr

  div.row
    div.col-md-6
      h5 Top Applications

      div#appsummary-container

    div.col-md-6
      h5 Top Window Titles

      div#windowtitles-container

      b-button(size="sm", variant="outline-secondary", v-on:click="numberOfWindowTitles += 5; queryWindowTitles()")
        icon(name="angle-double-down")
        | Show more

  hr

  h4 Timeline

  label.custom-control.custom-checkbox
    input.custom-control-input(type="checkbox", v-model="timelineShowAFK")
    span.custom-control-indicator
    span.custom-control-description
      | Show AFK time

  div#apptimeline-container

  hr

  h4 Clock

  b-alert(variant="warning" show)
    | #[b Note:] This is an early version. It has known issues that will be resolved in a future update.
    | See #[a(href="https://github.com/ActivityWatch/aw-webui/issues/36") issue #36] for details.

  aw-sunburst(:hierarchy="hierarchy")
</template>

<style lang="scss">

#apptimeline-container {
    white-space: nowrap;
    font-family: sans-serif;
    font-size: 11pt;
    line-height: 1.2em;
}

</style>

<script>
import Resources from '../resources.js';
import moment from 'moment';
import timeline from '../visualizations/timeline.js';
import summary from '../visualizations/summary.js';
import time from "../util/time.js";
import event_parsing from "../util/event_parsing.js";

import 'vue-awesome/icons/arrow-left'
import 'vue-awesome/icons/arrow-right'
import 'vue-awesome/icons/angle-double-down'
import 'vue-awesome/icons/refresh'

import Sunburst from '../visualizations/Sunburst.vue';

let $QueryView  = Resources.$QueryView;
let $CreateView  = Resources.$CreateView;
let $Info  = Resources.$Info;
let $Bucket = Resources.$Bucket;
let $Event  = Resources.$Event;

var daylength = 86400000;

export default {
  name: "Activity",
  data: () => {
    return {
      today: moment().format("YYYY-MM-DD"),

      filterAFK: true,
      timelineShowAFK: true,

      // Query variables
      duration: "",
      eventcount: 0,
      errormsg: "",
      numberOfWindowTitles: 5,

      // Sunburst stuff
      hierarchy: Object,
    }
  },

  components: {
    "aw-sunburst": Sunburst,
  },

  watch: {
    '$route': function(to, from) {
      console.log("Route changed")
      this.refresh();
    },
    'filterAFK': function(to, from) {
      this.refresh();
    },
    'timelineShowAFK': function(to, from) {
      this.refresh();
    },
  },

  computed: {
    readableDuration: function() { return time.seconds_to_duration(this.duration) },
    host: function() { return this.$route.params.host },
    date: function() { return this.$route.params.date || moment().format() },
    dateStart: function() { return moment(this.date).startOf('day').format() },
    dateShort: function() { return moment(this.date).format("YYYY-MM-DD") },
    windowBucketId: function() { return "aw-watcher-window_" + this.host },
    afkBucketId:    function() { return "aw-watcher-afk_"    + this.host }
  },

  mounted: function() {
    summary.create(document.getElementById("appsummary-container"));
    summary.create(document.getElementById("windowtitles-container"));
    timeline.create(document.getElementById("apptimeline-container"));

    this.refresh();
  },

  methods: {
    previousDay: function() { return moment(this.dateStart).subtract(1, 'days').format("YYYY-MM-DD") },
    nextDay: function() { return moment(this.dateStart).add(1, 'days').format("YYYY-MM-DD") },

    refresh: function() {
      this.query();
      this.visualize();
    },

    errorHandler: function(response) {
      console.error(response);
      this.errormsg = "Request error " + response.status + ". See F12 console for more info.";
    },

    query: function() {
      this.duration = "";
      this.eventcount = 0;
      this.errormsg = "";

      this.queryApps();
      this.queryWindowTitles();
      this.queryTimeline();
    },

    queryTimeline: function() {
      var timeline_view_name = "windowactivity_timeline@"+this.host;
      var query = this.windowTimelineQuery(this.windowBucketId, this.afkBucketId);
      $CreateView.save({viewname: timeline_view_name}, {'query': query}).then(
        (response) => { // Success
          if (response.status > 304){
            this.errorHandler(response);
          } else {
            this.queryViewTimeline(timeline_view_name);
          }
        }, this.errorHandler);
    },

    todaysEvents: function(bucket_id) {
      let today = moment(this.dateStart);
      let endOfToday = moment(today).add(1, "days");
      return $Event.get({id: bucket_id, limit: -1,
                         start: today.format(), end: endOfToday.format()})
                   .then((response) => response.json());
    },

    windowEventsFilteredByAFK: function() {
      return this.todaysEvents(this.windowBucketId)
        .then((events) => {
          return this.todaysEvents(this.afkBucketId).then((afkevents) => {
              if(this.filterAFK) {
                events = event_parsing.filterAFKTime(events, afkevents);
              }
              return events;
          })
        }, this.errorHandler);
    },

    groupAndSumEvents: function(events, groupingFunc) {
      let groups = _.groupBy(events, groupingFunc);
      let groupsList = _.values(groups);

      let summedEvents = _.map(groupsList,
            (v, i) => _.reduce(_.drop(v, 1),
                (acc, e) => {
                  acc.duration += e.duration;
                  delete acc.id;
                  delete acc.timestamp;
                  delete acc.range;
                  return acc;
                }, v[0]))

      // Sort objects by duration
      summedEvents = _.sortBy(summedEvents, (e) => e.duration).reverse();
      return summedEvents;
    },

    queryWindowTitles: function() {
      var container = document.getElementById("windowtitles-container")
      summary.set_status(container, "Loading...");

      this.windowEventsFilteredByAFK().then((events) => {
          let summedEvents = this.groupAndSumEvents(events, (o) => o.data.title);
          summedEvents = _.take(summedEvents, this.numberOfWindowTitles);
          summary.updateSummedEvents(container, summedEvents, (e) => e.data.title, (e) => e.data.app);
      });
    },

    queryApps: function(viewname){
      var container = document.getElementById("appsummary-container")
      summary.set_status(container, "Loading...");

      this.windowEventsFilteredByAFK().then((events) => {
          let summedEvents = this.groupAndSumEvents(events, (o) => o.data.app);
          summary.updateSummedEvents(container, summedEvents, (e) => e.data.app, (e) => e.data.app);
      });
    },

    queryViewTimeline: function(viewname){
      var timeline_elem = document.getElementById("apptimeline-container")
      timeline.set_status(timeline_elem, "Loading...");

      let today = moment(this.dateStart);
      let endOfToday = today.clone().add(1, 'days');
      $QueryView.get({"viewname": viewname,
                      "limit": -1,
                      "start": today.format(),
                      "end": endOfToday.format()})
        .then((response) => {
          if (response.status > 304){
            this.errorHandler(response);
          } else {
            var data = response.json();
            var eventlist = data["eventlist"];
            this.duration = data["duration"];
            this.eventcount = data["eventcount"]+this.eventcount;

            var apptimeline = event_parsing.parse_eventlist_by_apps(eventlist);
            var el = document.getElementById("apptimeline-container")
            timeline.update(el, apptimeline, this.duration, this.timelineShowAFK);
          }
        }, this.errorHandler);
    },


    windowTimelineQuery: function(windowbucket, afkbucket){
      return {
        'chunk': false,
        'cache': false,
        'transforms':
        [{
          'bucket': windowbucket,
          // TODO: How is this condition handled when bucket is cached? Seems to work...
          'filters': this.filterAFK ?
          [{
            'name': 'timeperiod_intersect',
            'transforms':
            [{
              'bucket': afkbucket,
              'filters':
              [{
                'name': 'include_keyvals',
                'key': 'status',
                'vals': ['not-afk'],
              }]
            }]
          }] : []
        }]
      };
    },

    // Everything below is related to the sunburst visualization
    getBucketInfo: function(bucket_id) {
      return $Bucket.get({"id": bucket_id}).then((response) => {
        return response.json();
      });
    },

    visualize: function() {
      function buildHierarchy(parents, children) {
          parents = _.sortBy(parents, "timestamp", "desc");
          children = _.sortBy(children, "timestamp", "desc");

          var i_child = 0;
          for(var i_parent = 0; i_parent < parents.length; i_parent++) {
              let p = parents[i_parent];
              let p_start = moment(p.timestamp);
              let p_end = p_start.clone().add(p.duration, "seconds");

              p.children = [];
              while(i_child < children.length) {
                  var e = children[i_child];
                  var e_start = moment(e.timestamp);
                  var e_end = e_start.clone().add(e.duration, "seconds");

                  let too_far = e_start.isAfter(p_end);
                  let before_parent = e_end.isBefore(p_start);
                  let within_parent = e_start.isAfter(p_start) && e_end.isBefore(p_end);
                  let after_parent = e_start.isAfter(p_end);

                  // TODO: This isn't correct, yet
                  if(before_parent) {
                    // Child is behind parent
                    //console.log("Too far behind: " + i_child);
                    i_child++;
                  } else if(within_parent) {
                    //console.log("Added relation: " + i_child);
                    p.children = _.concat(p.children, e);
                    i_child++;
                  } else if(after_parent) {
                    // Child is ahead of parent
                    //console.log("Too far ahead: " + i_child);
                    break;
                  } else {
                    // TODO: Split events when this happens
                    console.warn("Between parents");
                    p.children = _.concat(p.children, e);
                    i_child++;
                  }
              }
          }

          // Build the root node
          console.log(parents);
          let m_start = moment(_.first(parents).timestamp)
          let m_end = moment(_.tail(parents).timestamp)
          let duration = (m_end - m_start) / 1000;
          return {
            "timestamp": _.first(parents).timestamp,
            // TODO: If we want a 12/24h clock, this has to change
            "duration": duration,
            "data": {"title": "ROOT"},
            "children": parents
          }
      }

      function chunkHierarchy(events, key) {
          // TODO: Merge window events with same app and assign the title events as children
          let new_events = [events[0]];
          let p_i = 0;
          _.each(events, (e, i) => {
              if(e.data[key] === new_events[p_i].data[key]) {
                  //console.log("merge");
                  let e_moment = moment(e.timestamp);
                  let ne_moment = moment(new_events[p_i].timestamp);
                  new_events[p_i].duration = -e_moment.diff(ne_moment, "seconds", true) + e.duration;
                  console.log(new_events[p_i].duration);
              } else {
                  //console.log("skip");
                  //console.log(new_events[p_i].duration);
                  p_i++;
                  new_events[p_i] = e;
              }
          });
          _.each(new_events, (e, i) => {
              // Get rid of other keys
              e.data = _.pickBy(e.data, (v, k) => k === key);
          })
          return new_events;
      }

      function chunkHierarchy2(events, key) {
        events = _.sortBy(events, (e) => e.timestamp);
        events = _.reverse(events);
        events = _.reduce(events,
          function(acc, e) {
            let last = _.last(acc);
            if(last.data[key] === e.data[key]) {
              last.duration = moment(e.timestamp).diff(last.timestamp, "seconds", true) + e.duration;
            } else {
              acc.push(e);
            }
            return acc;
          },
          [events[0]]);
        return events;
      }

      this.todaysEvents(this.afkBucketId).then((events_afk) => {
        this.todaysEvents(this.windowBucketId).then((events_window) => {
          //events_afk = _.filter(events_afk, (e) => e.data.status == "not-afk");
          //events_window = _.filter(events_window, (e) => e.duration > 10);

          //events_afk = chunkHierarchy(events_afk, "status");
          //events_window = chunkHierarchy(events_window, "app");

          if(events_afk.length > 0 && events_window.length > 0) {
            this.hierarchy = buildHierarchy(events_afk, events_window);
          } else {
            // FIXME: This should do the equivalent of "No data" when such is the case, but it doesn't.
            console.log("HELLO");
            this.hierarchy = {
              "timestamp": "",
              // TODO: If we want a 12/24h clock, this has to change
              "duration": 0,
              "data": {"title": "ROOT"},
              "children": []
            };
          }
        });
      });
    },
  },
}
</script>
