<template>
  <section class="v-cal-content">
    <div class="v-cal-weekdays">
      <div class="v-cal-weekday-item">{{ activeDate.format('ddd MMMDo') }}</div>
    </div>
    <div class="v-cal-days scroll-box">
      <div class="v-cal-times">
        <div class="v-cal-hour all-day">{{ allDayLabel }}</div>
        <div
          class="v-cal-hour"
          :class="{ 'is-now': time.isSame(now, 'hour') }"
          v-for="time in times"
        >{{ time | formatTime(use12) }}</div>
      </div>
      <div class="v-cal-days__wrapper">
        <DayBase v-bind="activeViewProps"></DayBase>
      </div>
    </div>
  </section>
</template>

<script>
import moment, { min } from "moment";
import Event from "../../model/Event";
import { EventBus } from "../EventBus";
import ShowsTimes from "../mixins/ShowsTimes";
import IsView from "../mixins/IsView";
import DayBase from "./DayBase";

export default {
  name: "day",
  components: { DayBase },
  mixins: [IsView, ShowsTimes],
  computed: {
    // newEvents() {
    //   // return this.events.map(e => {
    //   //   return new Event(e).bindGetter("displayText", this.eventDisplay);
    //   // });
    //   return this.events;
    // },

    activeViewProps() {
      let props = {
        activeDate: this.activeDate,
        minDate: this.minDate,
        maxDate: this.maxDate,
        use12: this.use12,
        events: this.events.filter( event => {
          console.log(this.activeDate, this.activeView);
            const same = event.date.isSame(this.activeDate, 'day');
            return same;
        })
      };

      props.timeRange = this.timeRange;
      props.showTimeMarker = this.showTimeMarker;

      return props;
    }
  },
  methods: {
    buildCalendar() {}
  }
};
</script>

<style scoped>
</style>
