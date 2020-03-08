<template>
  <div class="calendar-body">
    <div class="v-cal is-fullheight padding-t0px">
        <component
                :is="activeView"
                class="v-cal-row"
                :class="'v-cal-content--' + activeView"
                v-bind="activeViewProps"
        ></component>
    </div>
  </div>
</template>

<script>
    import Event from '../model/Event';
    import config from '../utils/config';
    import { defaultLabels, defaultViews } from '../utils/config';
    import { EventBus } from './EventBus';

    import moment from 'moment';
    import Month from './views/Month';
    import Week from './views/Week';
    import Day from './views/Day';

    export default {
        name: "VueScheduler",
        components: { Month, Week, Day },
        props: {
            events: {
                type: Array,
                default: () => []
            },
            minDate: {
                type: [Date, Object],
                default: () => config.minDate
            },
            maxDate: {
                type: [Date, Object],
                default: () => config.maxDate
            },
            labels: {
                type: Object,
                default: () => config.labels,
                validator(value) {
                    for (const labelKey in defaultLabels ) {
                        if ( !value.hasOwnProperty(labelKey) ) {
                            console.error('Missing prop label: ' + labelKey);
                            return false;
                        }
                    }
                    return true;
                }
            },
            timeRange: {
                type: Array,
                default: () => config.timeRange,
                validator(value) {
                    if ( value.length !== 2 || value[0] > value[1] || value[0] < 0 || value[1] > 23) {
                        console.error('Invalid time range.');
                        return false;
                    }
                    return true;
                }
            },
            availableViews: {
                type: Array,
                default: () => config.availableViews,
                validator (value) {
                    const possible = defaultViews;
                    let error = false;
                    value.forEach(view => {
                        if ( possible.indexOf(view) === -1 ) {
                            console.error('Invalid view: ' + view);
                            error = true;
                        }
                    });
                    return !error;
                }
            },
            initialDate: {
                type: [Date, Object],
                default: () => config.initialDate
            },
            initialView: {
                type: String,
                default: () => config.initialView
            },
            use12: {
                type: Boolean,
                default: () => config.use12
            },
            showTimeMarker: {
                type: Boolean,
                default: () => config.showTimeMarker
            },
            eventDisplay: {
                type: [String, Function],
                default: () => config.eventDisplay
            },
            disableDialog: {
                type: Boolean,
                default: false
            },
            eventDialogConfig: {
                type: Object,
                default: () => { return {} }
            }
        },
        data() {
            return {
                today: moment(),
                activeView: '',
                activeDate: null
            }
        },
        mounted() {
            //  Initial setup
            this.activeView = this.initialView;
            this.activeDate = moment(this.initialDate);
            //  Bind events
            this.bindEvents();
        },
        beforeDestroy() {
            //EventBus.$off('day-clicked');
            EventBus.$off('range-selected');
            EventBus.$off('range-select-changed');
            EventBus.$off('event-clicked');
        },
        methods: {
            bindEvents() {
                EventBus.$on('range-selected', (data) => {
                    this.$emit('range-selected', data);
                });

                EventBus.$on('range-select-changed', (data) => {
                    this.$emit('range-select-changed', data);
                });
                EventBus.$on('event-clicked', (event) => {
                    this.$emit('event-clicked', event._e);
                });
            }
        },
        watch: {
            initialDate() {
                this.activeDate = moment(this.initialDate);
            },
            initialView() {
                this.activeView = this.initialView;
            },
            activeDate() {
                this.$emit(this.activeView + '-changed', this.activeDate.toDate() );
            },
            activeView() {
                this.$emit('view-changed', this.activeView);
            }
        },
        computed: {
            newEvents() {
                return this.events;
            },
            activeViewProps() {
                let props = {
                    activeDate: this.activeDate,
                    minDate: this.minDate,
                    maxDate: this.maxDate,
                    use12: this.use12,
                    events: this.events.filter( event => {
                        return event.date.isSame(this.activeDate, this.activeView);
                    })
                };

                if ( this.activeView === 'week' || this.activeView === 'day') {
                    props.allDayLabel = this.labels.all_day;
                    props.timeRange = this.timeRange;
                    props.showTimeMarker = this.showTimeMarker;
                }
                return props;
            }
        }
    }
</script>

<style lang="scss">
@import '../scss/main.scss';
</style>
