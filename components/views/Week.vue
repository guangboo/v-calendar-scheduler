<template>
  <section class="v-cal-content">
    <div class="v-cal-weekdays">
      <div class="v-cal-weekdays__padding">
        <div class="v-cal-times">
          <!--Fake, hidden time-->
          <div class="v-cal-hour">00:00 <template v-if="use12">PM</template></div>
        </div>
      </div>
      <div class="v-cal-weekday__wrapper">
        <div class="v-cal-weekday-item" v-for="day in days">{{ day.d.format('ddd MMMDo') }}</div>
      </div>
    </div>
    <div class="v-cal-days scroll-box">
      <div class="v-cal-times">
        <div class="v-cal-hour all-day">{{ allDayLabel }}</div>
        <div class="v-cal-hour" :class="{ 'is-now': time.isSame(now, 'hour') }" v-for="time in times">{{ time | formatTime(use12) }}</div>
      </div>
      <div class="v-cal-days__wrapper">
          <DayBase v-for="day in days" :class="{ 'is-today': day.isToday, 'is-disabled': day.isDisabled }" v-bind="activeViewProps(day)"></DayBase>
      </div>
    </div>
  </section>
</template>

<script>
    import moment from 'moment';
    import { EventBus } from '../EventBus';
    import EventItem from '../EventItem';
    import IsView from '../mixins/IsView';
    import ShowsTimes from '../mixins/ShowsTimes';
    import DayBase from './DayBase';

    export default {
        name: "week",
        mixins: [ IsView, ShowsTimes ],
        components: { EventItem, DayBase },
        data() {
            return {
                days: [],
                // newEvents: JSON.parse(JSON.stringify(this.events))
            }
        },
        mounted() {
            this.buildCalendar();
        },
        methods: {
            timeClicked(data) {
                EventBus.$emit('range-selected', data)
            },

            activeViewProps(day) {
                let props = {
                    activeDate: day.d,
                    minDate: this.minDate,
                    maxDate: this.maxDate,
                    use12: this.use12,
                    events: day.events
                };

                props.timeRange = this.timeRange;
                props.showTimeMarker = this.showTimeMarker;
                
                return props;
            },
            
            buildCalendar() {
                //  Reset events
                // this.newEvents = JSON.parse(JSON.stringify(this.events));

                this.days = [];

                let now = moment();

                let temp = moment( this.activeDate ).day(moment.localeData().firstDayOfWeek());
                let w = temp.week();

                this.days = [];

                do {
                    const day = moment(temp);

                    const dayEvents = this.events.filter( e => e.date.isSame(day, 'day') )
                        .sort( (a, b) => {
                            if ( !a.startTime ) return -1;
                            if ( !b.startTime ) return 1;
                            return moment(a.startTime).format('HH') - moment(b.startTime).format('HH');
                        });
                    const mappedEvents = dayEvents.map( event => {
                        event.overlaps = dayEvents.filter( e => moment(event.startTime).isBetween( moment(e.startTime), moment(e.endTime) ) && e !== event ).length;
                        return event;
                    });

                    let newDay = {
                        d: day,
                        isPast: temp.isBefore( now, 'day' ),
                        isToday: temp.isSame( now, 'day' ),
                        isDisabled: this.isDayDisabled(temp),
                        availableTimes: this.times.map( time => moment(time).dayOfYear( day.dayOfYear() ) ),
                        events: mappedEvents
                    };
                    this.days.push(newDay);

                    temp.add( 1, 'day' );
                } while ( temp.week() === w );

            }
        },
        watch: {
            timeRange() {
                this.buildCalendar();
            }
        }
    }
</script>

<style scoped>

</style>
