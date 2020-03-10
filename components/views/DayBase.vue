<template>
  <div
    class="v-cal-day v-cal-day--day"
    ref="refDays"
    @mousedown="selectStart($event)"
    @mousemove="selectMove($event)"
    @mouseup="selectDone($event)"
    :class="{ 'is-today': day.isToday }"
    v-if="day !== null"
  >
    <!-- for all day -->
    <div class="v-cal-day__hour-block" ref="refBase">
      <span class="v-cal-day__hour-block-fill">
        00:00
        <template v-if="use12">PM</template>
      </span>
      <div class="v-cal-day__hour-content">
        <div
          class="v-cal-event-list"
          :class="{'tiny-events': day.events.filter(e => !e.startTime).length > 2}"
        >
          <event-item
            v-for="event,index in day.events.filter(e => !e.startTime)"
            :key="index"
            :event="event"
            :use12="use12"
            @click.prevent="eventBus.$emit('event-clicked', event)"
          ></event-item>
        </div>
      </div>
    </div>
    <div
      class="v-cal-event-item drawer"
      v-if="startTime"
      ref="refDrawer"
      :style="{top: selectionTop + 'px', display: 'block', height: selectionHeight + 'px'}"
      @mousedown="startMove($event)"
      @mousemove="selectMove($event)"
      @mouseup="selectDone($event)"
    >
      <span
        class="scheduler-time"
      >{{ selectionStartTime | formatTime(use12) }} ~ {{ selectionEndTime | formatTime(use12) }}</span>
    </div>

    <!-- for each time -->
    <div
      class="v-cal-day__hour-block"
      @mousedown="selectStart($event)"
      :class="[ time.isSame(now, 'hour') ? 'is-now' : '', hourClass ]"
      v-for="time in day.availableTimes"
    >
      <span class="v-cal-day__hour-block-fill">{{ time | formatTime(use12) }}</span>
      <div class="v-cal-day__hour-content">
        <div class="v-cal-event-list">
          <event-item
            v-for="event, index in day.events"
            v-if="event.startTime && time.hours() === event.startTime.hours()"
            :key="index"
            :event="event"
            :use12="use12"
          ></event-item>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import moment, { min } from "moment";
import { EventBus } from "../EventBus";
import Event from "../../model/Event";
import EventItem from "../EventItem";
import IsView from "../mixins/IsView";
import ShowsTimes from "../mixins/ShowsTimes";
//import $ from "jquery";

export default {
  name: "daybase",
  mixins: [IsView, ShowsTimes],
  components: { EventItem },
  data() {
    return {
      day: null,
      mode: null,
      startTime: null,
      endTime: null,

      oStartTime: null,
      oEndTime: null,
      oPageY: null
    };
  },
  mounted() {
    this.buildCalendar();
  },
  computed: {
    selectionStartTime() {
      if (this.startTime.isBefore(this.endTime))
        return this.alignFloorTime(this.startTime);
      else return this.alignFloorTime(this.endTime);
    },
    selectionEndTime() {
      if (this.startTime.isAfter(this.endTime))
        return this.alignFloorTime(this.startTime);
      else return this.alignFloorTime(this.endTime);
    },
    originalTop() {
      if (this.startTime) {
        if (this.startTime === this.endTime) {
          return 0;
        }
        const top = this.time2Position({
          hour: this.startTime.hour(),
          minute: this.startTime.minute()
        });
        return top;
      }
    },
    selectionTop() {
      if (this.startTime) {
        if (
          this.startTime === this.endTime &&
          this.startTime.hour() === 0 &&
          this.startTime.minute() === 0
        ) {
          return 0;
        }
        if (this.startTime.isBefore(this.endTime))
          return this.time2Position({
            hour: this.startTime.hour(),
            minute: this.startTime.minute()
          });
        else
          return this.time2Position({
            hour: this.endTime.hour(),
            minute: this.endTime.minute()
          });
      }
    },
    selectionHeight() {
      if (this.startTime) {
        if (
          this.startTime === this.endTime &&
          this.startTime.hour() === 0 &&
          this.startTime.minute() === 0
        ) {
          return 60;
        }
        const height = Math.abs(this.startTime - this.endTime) / 1000 / 60;
        return height;
      }
    }
  },
  watch: {
    day(val) {
      this.startTime = this.endTime = null;
      this.mode = null;
    }
  },
  methods: {
    alignFloorTime(time) {
      let m = time.minute();
      m = m - Math.floor(m / 15) * 15;
      time.subtract(m, "minutes");
      return time;
    },

    alignTopTime(currentY) {
      const base = 0;
      const timeData = (currentY - base) / 60 - 1;
      const hour = Math.floor(timeData);
      const minuteData = (timeData - hour) * 60;
      let minute = Math.floor(minuteData / 15) * 15;
      return this.toMoment({ hour, minute });
    },

    getCurrentTime(event) {
      let hour = 0,
        minute = 0;
      const currentY = this.getCurrentY(event);

      if (currentY <= 0) {
        return this.toMoment({ hour, minute });
      }

      const base = 0;
      const timeData = (currentY - base) / 60 - 1;
      hour = Math.floor(timeData);
      const minuteData = (timeData - hour) * 60;
      minute = Math.floor(minuteData);

      let momentTime;

      if (hour === 24 || timeData > 23.9) {
        hour = 0;
        minute = 0;
        momentTime = this.toMoment({ hour, minute });
        momentTime.add(1, "days");
      } else {
        momentTime = this.toMoment({ hour, minute });
      }

      return momentTime;
    },

    time2Position(time, offsetMinutes) {
      const base = 0;
      return base + (time.hour + 1) * 60 + time.minute + (offsetMinutes || 0);
    },

    checkBoundary(pos) {
      const topBoundary = this.$refs.refBase.offsetTop + 60;
      const bottomBoundary = this.$refs.refDays.offsetHeight + 15;

      if (pos < topBoundary) return topBoundary;
      else if (pos >= bottomBoundary) return bottomBoundary;
      return pos;
    },

    toMoment(hm) {
      const date = this.day.d.toArray();
      return moment([date[0], date[1], date[2], hm.hour, hm.minute]);
    },

    alignTop(offsetY, pageY) {
      pageY = pageY || offsetY;
      const alignOffset = offsetY % 15;
      return this.checkBoundary(pageY - alignOffset);
    },

    getCurrentY(event) {
      return (
        event.pageY +
        window.document.querySelector(".calendar-body").scrollTop -
        this.$refs.refBase.offsetParent.offsetTop
      );
    },

    selected(event) {
      let startTime = this.selectionStartTime;
      let endTime = this.selectionEndTime;

      this.startTime = moment(startTime);
      this.endTime = moment(endTime);

      this.emitEvent("range-selected", this.startTime, this.endTime);

      this.mode = null;
    },

    moved(event) {
      let startTime = this.selectionStartTime;
      let endTime = this.selectionEndTime;

      this.startTime = moment(startTime);
      this.endTime = moment(endTime);

      if (this.startTime != this.oStartTime || this.endTime !== this.oEndTime) {
        this.emitEvent("range-select-changed", this.startTime, this.endTime);
      }

      this.oStartTime = this.oEndTime = this.oPageY = null;
      this.$refs.refDrawer.style.cursor = "pointer";
      this.mode = null;
    },

    resized(event) {
      let startTime = this.selectionStartTime;
      let endTime = this.selectionEndTime;

      this.startTime = moment(startTime);
      this.endTime = moment(endTime);

      this.emitEvent("range-select-changed", this.startTime, this.endTime);

      this.mode = null;
    },

    startMove(event) {
      const pageY = this.getCurrentY(event);
      if (pageY < 60) {
        return;
      }
      if (this.mode === "select") {
        this.selected(event);
      } else {
        const time = this.getCurrentTime(event);
        const pageY = this.getCurrentY(event);
        if (Math.abs(time - this.endTime) / 1000 / 60 <= 8) {
          this.mode = "resize-s";
          this.$refs.refDrawer.style.cursor = "ns-resize";
        } else {
          this.mode = "move";
          this.$refs.refDrawer.style.cursor = "move";
          this.oStartTime = this.startTime;
          this.oEndTime = this.endTime;
          this.oPageY = pageY;
        }

        const self = this;
        const f = function(event) {
          if (self.mode === "resize-s") {
            self.resized(event);
          } else if (self.mode === "move") {
            self.moved(event);
          }
          window.document.body.removeEventListener("mouseup", f);
        };

        window.document.body.addEventListener("mouseup", f);
      }
    },

    selectStart(event) {
      if (/v-cal-event/.test(event.target.className)) return;

      if (this.mode == null) {
        const pageY = this.getCurrentY(event);
        if (pageY < 60) {
          const date = this.day.d.toArray();
          this.startTime = this.endTime = moment([date[0], date[1], date[2]]);
          this.emitEvent("range-selected", this.startTime, this.endTime);
          return;
        }

        const currentTime = this.alignTopTime(pageY);

        this.startTime = currentTime;
        this.endTime = moment(currentTime);
        this.endTime.add(30, "minutes");
        this.mode = "select";

        const self = this;
        const f = function(event) {
          if (self.mode === "select") {
            self.selected(event);
            window.document.body.removeEventListener("mouseup", f);
          }
        };

        window.document.body.addEventListener("mouseup", f);
      }
    },
    selectMove(event) {
      const pageY = this.getCurrentY(event);
      if (this.mode === "select") {
        const pageY = this.getCurrentY(event);
        const currentTime = this.getCurrentTime(event);

        if (!currentTime.isValid() || pageY < 0) {
          this.endTime = this.toMoment({ hour: 0, minute: 0 });
        } else {
          this.endTime = this.getCurrentTime(event);
        }
      } else if (
        this.mode === "move" &&
        this.startTime &&
        /drawer/.test(event.target.className)
      ) {
        const lastStartTime = this.oStartTime;
        const lastEndTime = this.oEndTime;
        const lastPageY = this.oPageY;

        let offset = pageY - lastPageY;
        let startTime = moment(lastStartTime);
        let endTime = moment(lastEndTime);
        const today = this.toMoment({ hour: 0, minute: 0 });
        const tommorow = moment(today);
        tommorow.add(1, "days");

        startTime.add(offset, "minutes");
        endTime.add(offset, "minutes");

        if (today > startTime) {
          startTime = today;
          endTime = moment(startTime + (lastEndTime - lastStartTime));
        } else if (endTime > tommorow) {
          endTime = tommorow;
          startTime = moment(endTime - (lastEndTime - lastStartTime));
        }
        this.startTime = startTime;
        this.endTime = endTime;
      } else if (this.mode === "resize-s") {
        let currentTime = this.getCurrentTime(event);
        this.endTime = currentTime;
      } else if (
        !this.mode &&
        this.startTime &&
        /drawer/.test(event.target.className)
      ) {
        let time = this.getCurrentTime(event);

        if (Math.abs(time - this.endTime) / 1000 / 60 <= 8) {
          this.$refs.refDrawer.style.cursor = "ns-resize";
        } else {
          this.$refs.refDrawer.style.cursor = "move";
        }
      }
    },
    selectDone(event, day, time) {
      if (this.mode === "select") {
        this.selected(event);
      } else if (this.mode === "move") {
        this.moved(event);
      } else if (this.mode === "resize-s") {
        this.resized(event);
      }
      if (this.$refs.refDrawer) {
        this.$refs.refDrawer.style.cursor = "pointer";
      }
    },
    timeClicked(data) {
      const date = this.day.d.toArray();
      this.startTime = this.endTime = moment([date[0], date[1], date[2]]);

      this.emitEvent("range-selected", this.startTime, this.endTime);
    },

    emitEvent(eventName, startTime, endTime) {
      const eventObject = {
        startTime,
        endTime,
        reject() {},
        accept() {}
      };
      EventBus.$emit("range-selected", eventObject);
    },

    buildCalendar() {
      let now = moment();

      const today = moment(this.activeDate);

      const dayEvents = this.events
        .filter(e => moment(e.date).isSame(today, "day"))
        .sort((a, b) => {
          if (!a.startTime) return -1;
          if (!b.startTime) return 1;
          return (
            moment(a.startTime, "HH:mm").format("HH") -
            moment(b.startTime, "HH:mm").format("HH")
          );
        });
      const mappedEvents = dayEvents.map(event => {
        event.overlaps = dayEvents.filter(
          e =>
            moment(event.startTime, "HH:mm").isBetween(
              moment(e.startTime, "HH:mm"),
              moment(e.endTime, "HH:mm")
            ) && e !== event
        ).length;
        return event;
      });

      this.day = {
        d: today,
        isPast: today.isBefore(now, "day"),
        isToday: today.isSame(now, "day"),
        availableTimes: this.times,
        events: mappedEvents
      };
    }
  }
};
</script>