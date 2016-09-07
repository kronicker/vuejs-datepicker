<template>
  <div class="datepicker">
    <input
        type="{{inline ? 'hidden' : 'text'}}"
        class=""
        name="{{ name }}"
        @click="showCalendar"
        value="{{ formattedValue }}"
        placeholder="{{ placeholder }}"
        readonly>

        <!-- Day View -->
        <div class="calendar" v-show="showDayView" v-bind:style="calendarStyle">
            <header>
                <span
                    @click="previousMonth"
                    class="prev"
                    v-bind:class="{ 'disabled' : previousMonthDisabled(currDate) }">&lt;</span>
                <span @click="showMonthCalendar" class="up">{{ currMonthName }} {{ currYear }}</span>
                <span
                    @click="nextMonth"
                    class="next"
                    v-bind:class="{ 'disabled' : nextMonthDisabled(currDate) }">&gt;</span>
            </header>
            <span class="cell day-header" v-for="d in translation.days">{{ d }}</span>
            <span class="cell day blank" v-for="d in blankDays"></span><!--
            --><span class="cell day"
                v-for="day in days"
                track-by="$index"
                v-bind:class="{ 'selected':day.isSelected, 'disabled':day.isDisabled }"
                @click="selectDate(day)">{{ day.date }}</span>

        </div>

        <!-- Month View -->
        <div class="calendar" v-show="showMonthView" v-bind:style="calendarStyleSecondary">
            <header>
                <span
                    @click="previousYear"
                    class="prev"
                    v-bind:class="{ 'disabled' : previousYearDisabled(currDate) }">&lt;</span>
                <span @click="showYearCalendar" class="up">{{ getYear() }}</span>
                <span
                    @click="nextYear"
                    class="next"
                    v-bind:class="{ 'disabled' : nextYearDisabled(currDate) }">&gt;</span>
            </header>
            <span class="cell month"
                v-for="month in months"
                track-by="$index"
                v-bind:class="{ 'selected': month.isSelected, 'disabled': month.isDisabled }"
                @click.stop="selectMonth(month)">{{ month.month }}</span>
        </div>

        <!-- Year View -->
        <div class="calendar" v-show="showYearView" v-bind:style="calendarStyleSecondary">
            <header>
                <span @click="previousDecade" class="prev"
                    v-bind:class="{ 'disabled' : previousDecadeDisabled(currDate) }">&lt;</span>
                <span>{{ getDecade() }}</span>
                <span @click="nextDecade" class="next"
                    v-bind:class="{ 'disabled' : nextMonthDisabled(currDate) }">&gt;</span>
            </header>
            <span
                class="cell year"
                v-for="year in years"
                track-by="$index"
                v-bind:class="{ 'selected': year.isSelected, 'disabled': year.isDisabled }"
                @click.stop="selectYear(year)">{{ year.year }}</span>
        </div>

  </div>
</template>

<script>
import DateUtils from './utils/DateUtils.js'
import DateLanguages from './utils/DateLanguages.js'

/**
 * TODO
 * support disabled dates array
 */
export default {

    props: {
        value: {
            validator: function(val) {
                return val === null || val instanceof Date
            }
        },
        name: {
            value: String
        },
        format: {
            value: String,
            default: 'dd MMM yyyy'
        },
        language: {
            value: String,
            default: 'en'
        },
        disabled: {
            type: Object
        },
        placeholder: {
            type: String
        },
        inline: {
            type: Boolean
        }
    },
    data() {
        return {
            /*
             * Vue cannot observe changes to a Date Object so date must be stored as a timestamp
             * {Number}
             */
            currDate: new Date().getTime(),
            /*
             * Selected Date
             * {Date}
             */
            selectedDate: null,
            /**
             * Flags to show calendar views
             * {Boolean}
             */
            showDayView: false,
            showMonthView: false,
            showYearView: false,
            /**
             * Helper arrays for names
             */
            translation: DateLanguages.translations[this.language],
            formattedValue: null,
            /**
             * Positioning
             */
            calendarHeight: 0
        }
    },
    watch: {
        value(value) {
            this.setValue(value)
        }
    },
    computed: {
        currMonthName() {
            const d = new Date(this.currDate)
            return DateUtils.getMonthNameAbbr(d.getMonth(), this.translation.months.abbr)
        },
        currYear() {
            const d = new Date(this.currDate)
            return d.getFullYear()
        },
        /**
         * Returns the day number of the week less one for the first of the current month
         * Used to show amount of empty cells before the first in the day calendar layout
         * @return {Number}
         */
        blankDays() {
            const d = new Date(this.currDate)
            let dObj = new Date(d.getFullYear(), d.getMonth(), 1, d.getHours(), d.getMinutes())
            return dObj.getDay() - 1
        },
        days() {
            const d = new Date(this.currDate)
            let days = []
            // set up a new date object to the beginning of the current 'page'
            let dObj = new Date(d.getFullYear(), d.getMonth(), 1, d.getHours(), d.getMinutes())
            let daysInMonth = DateUtils.daysInMonth(dObj.getFullYear(), dObj.getMonth())
            for (let i = 0; i < daysInMonth; i++) {
                days.push({
                    date: dObj.getDate(),
                    timestamp: dObj.getTime(),
                    isSelected: this.isSelectedDate(dObj),
                    isDisabled: this.isDisabledDate(dObj)
                })
                dObj.setDate(dObj.getDate() + 1)
            }
            return days
        },
        months() {
            const d = new Date(this.currDate)
            let months = []
            // set up a new date object to the beginning of the current 'page'
            let dObj = new Date(d.getFullYear(), 0, d.getDate(), d.getHours(), d.getMinutes())
            for (let i = 0; i < 12; i++) {
                months.push({
                    month: DateUtils.getMonthName(i, this.translation.months.original),
                    timestamp: dObj.getTime(),
                    isSelected: this.isSelectedMonth(dObj),
                    isDisabled: this.isDisabledMonth(dObj)
                })
                dObj.setMonth(dObj.getMonth() + 1)
            }
            return months
        },
        years() {
            const d = new Date(this.currDate)
            let years = []
            // set up a new date object to the beginning of the current 'page'
            let dObj = new Date(Math.floor(d.getFullYear() / 10) * 10, d.getMonth(), d.getDate(), d.getHours(), d.getMinutes())
            for (let i = 0; i < 10; i++) {
                years.push({
                    year: dObj.getFullYear(),
                    timestamp: dObj.getTime(),
                    isSelected: this.isSelectedYear(dObj),
                    isDisabled: this.isDisabledYear(dObj)
                })
                dObj.setFullYear(dObj.getFullYear() + 1)
            }
            return years
        },
        calendarStyle() {
            let elSize = this.$el.getBoundingClientRect()
            let heightNeeded = elSize.top + elSize.height + this.calendarHeight
            let styles = {}
            // if the calendar doesn't fit on the window without scrolling position it above the input
            if (heightNeeded > window.innerHeight) {
                styles = {
                    'bottom': elSize.height + 'px'
                }
            }
            if (this.isInline()) {
                styles.position = 'static'
            }

            return styles
        },
        calendarStyleSecondary() {
            return (this.isInline()) ? { 'position': 'static' } : {}
        }
    },
    methods: {
        close() {
            this.showDayView = this.showMonthView = this.showYearView = false
        },
        isOpen() {
            return this.showDayView || this.showMonthView || this.showYearView
        },
        isInline() {
            return typeof this.inline !== 'undefined' && this.inline
        },
        showCalendar() {
            if (this.isInline()) {
                return
            }
            (this.isOpen()) ? this.close() : this.showDayCalendar()
        },
        showDayCalendar() {
            this.close()
            this.showDayView = true
            // set the calendar height property once is has been rendered
            this.$nextTick(function() {
                this.calendarHeight = this.$el.querySelector('.calendar').getBoundingClientRect().height
            })
        },
        showMonthCalendar() {
            this.close()
            this.showMonthView = true
        },
        showYearCalendar() {
            this.close()
            this.showYearView = true
        },

        setDate(timestamp) {
            this.value = this.selectedDate = new Date(timestamp)
            this.currDate = timestamp

            let d = new Date(timestamp)
            this.formattedValue = DateUtils.formatDate(d, this.format, this.translation)

            this.dispatchEvent(timestamp)
        },

        dispatchEvent(timestamp) {
            if (this.name) {
                this.$dispatch('datepicker.' + this.name, {
                    value: timestamp
                })
            }
        },

        selectDate(day) {
            if (day.isDisabled) {
                return false
            }
            this.setDate(day.timestamp)
            this.isInline() ? this.showDayCalendar() : this.close()
        },

        selectMonth(month) {
            if (month.isDisabled) {
                return false
            }
            this.currDate = month.timestamp
            this.showDayCalendar()
        },

        selectYear(year) {
            if (year.isDisabled) {
                return false
            }
            this.currDate = year.timestamp
            this.showMonthCalendar()
        },

        getYear() {
            let d = new Date(this.currDate)
            return d.getFullYear()
        },
        getDecade() {
            let d = new Date(this.currDate)
            let sD = Math.floor(d.getFullYear() / 10) * 10
            return sD + '\'s'
        },

        previousMonth() {
            if (this.previousMonthDisabled()) {
                return false
            }
            let d = new Date(this.currDate)
            d.setMonth(d.getMonth() - 1)
            this.currDate = d.getTime()
        },
        previousMonthDisabled() {
            if (typeof this.disabled === 'undefined' || typeof this.disabled.to === 'undefined' || !this.disabled.to) {
                return false
            }
            let d = new Date(this.currDate)
            if (
                this.disabled.to.getMonth() >= d.getMonth() &&
                this.disabled.to.getFullYear() >= d.getFullYear()
            ) {
                return true
            }
            return false
        },
        nextMonth() {
            if (this.nextMonthDisabled()) {
                return false
            }
            let d = new Date(this.currDate)
            d.setMonth(d.getMonth() + 1)
            this.currDate = d.getTime()
        },
        nextMonthDisabled() {
            if (typeof this.disabled === 'undefined' || typeof this.disabled.from === 'undefined' || !this.disabled.from) {
                return false
            }
            let d = new Date(this.currDate)
            if (
                this.disabled.from.getMonth() <= d.getMonth() &&
                this.disabled.from.getFullYear() <= d.getFullYear()
            ) {
                return true
            }
            return false
        },
        previousYear() {
            if (this.previousYearDisabled()) {
                return false
            }
            let d = new Date(this.currDate)
            d.setYear(d.getFullYear() - 1)
            this.currDate = d.getTime()
        },
        previousYearDisabled() {
            if (typeof this.disabled === 'undefined' || typeof this.disabled.to === 'undefined' || !this.disabled.to) {
                return false
            }
            let d = new Date(this.currDate)
            if (this.disabled.to.getFullYear() >= d.getFullYear()) {
                return true
            }
            return false
        },
        nextYear() {
            if (this.nextYearDisabled()) {
                return false
            }
            let d = new Date(this.currDate)
            d.setYear(d.getFullYear() + 1)
            this.currDate = d.getTime()
        },
        nextYearDisabled() {
            if (typeof this.disabled === 'undefined' || typeof this.disabled.from === 'undefined' || !this.disabled.from) {
                return false
            }
            let d = new Date(this.currDate)
            if (this.disabled.from.getFullYear() <= d.getFullYear()) {
                return true
            }
            return false
        },

        previousDecade() {
            if (this.previousDecadeDisabled()) {
                return false
            }
            let d = new Date(this.currDate)
            d.setYear(d.getFullYear() - 10)
            this.currDate = d.getTime()
        },
        previousDecadeDisabled() {
            if (typeof this.disabled === 'undefined' || typeof this.disabled.to === 'undefined' || !this.disabled.to) {
                return false
            }
            let d = new Date(this.currDate)
            if (Math.floor(this.disabled.to.getFullYear() / 10) * 10 <= Math.floor(d.getFullYear() / 10) * 10) {
                return true
            }
            return false
        },
        nextDecade() {
            if (this.nextDecadeDisabled()) {
                return false
            }
            let d = new Date(this.currDate)
            d.setYear(d.getFullYear() + 10)
            this.currDate = d.getTime()
        },
        nextDecadeDisabled() {
            if (typeof this.disabled === 'undefined' || typeof this.disabled.from === 'undefined' || !this.disabled.from) {
                return false
            }
            let d = new Date(this.currDate)
            if (Math.ceil(this.disabled.from.getFullYear() / 10) * 10 >= Math.ceil(d.getFullYear() / 10) * 10) {
                return true
            }
            return false
        },

        /**
         * Whether a day is selected
         * @param {Date}
         * @return {Boolean}
         */
        isSelectedDate(dObj) {
            return this.selectedDate && this.selectedDate.toDateString() === dObj.toDateString()
        },

        /**
         * Whether a day is disabled
         * @param {Date}
         * @return {Boolean}
         */
        isDisabledDate(date) {
            let disabled = false

            if (typeof this.disabled === 'undefined') {
                return false
            }

            if (typeof this.disabled.dates !== 'undefined') {
                this.disabled.dates.forEach(function(d) {
                    if (date.toDateString() === d.toDateString()) {
                        disabled = true
                        return true
                    }
                })
            }
            if (typeof this.disabled.to !== 'undefined' && this.disabled.to && date < this.disabled.to) {
                disabled = true
            }
            if (typeof this.disabled.from !== 'undefined' && this.disabled.from && date > this.disabled.from) {
                disabled = true
            }
            if (typeof this.disabled.days !== 'undefined' && this.disabled.days.indexOf(date.getDay()) !== -1) {
                disabled = true
            }
            return disabled
        },

        /**
         * Whether the selected date is in this month
         * @param {Date}
         * @return {Boolean}
         */
        isSelectedMonth(date) {
            return (this.selectedDate &&
                this.selectedDate.getFullYear() === date.getFullYear() &&
                this.selectedDate.getMonth() === date.getMonth())
        },

        /**
         * Whether a month is disabled
         * @param {Date}
         * @return {Boolean}
         */
        isDisabledMonth(date) {
            let disabled = false

            if (typeof this.disabled === 'undefined') {
                return false
            }

            if (typeof this.disabled.to !== 'undefined' && this.disabled.to) {
                if (
                    (date.getMonth() < this.disabled.to.getMonth() && date.getFullYear() <= this.disabled.to.getFullYear()) ||
                    date.getFullYear() < this.disabled.to.getFullYear()
                ) {
                    disabled = true
                }
            }
            if (typeof this.disabled.from !== 'undefined' && this.disabled.from) {
                if (
                    this.disabled.from &&
                    (date.getMonth() > this.disabled.from.getMonth() && date.getFullYear() >= this.disabled.from.getFullYear()) ||
                    date.getFullYear() > this.disabled.from.getFullYear()
                ) {
                    disabled = true
                }
            }
            return disabled
        },

        /**
         * Whether a year is disabled
         * @param {Date}
         * @return {Boolean}
         */
        isSelectedYear(date) {
            return this.selectedDate && this.selectedDate.getFullYear() === date.getFullYear()
        },

        /**
         * Whether a month is disabled
         * @param {Date}
         * @return {Boolean}
         */
        isDisabledYear(date) {
            let disabled = false
            if (typeof this.disabled === 'undefined' || !this.disabled) {
                return false
            }

            if (typeof this.disabled.to !== 'undefined' && this.disabled.to) {
                if (date.getFullYear() < this.disabled.to.getFullYear()) {
                    disabled = true
                }
            }
            if (typeof this.disabled.from !== 'undefined' && this.disabled.from) {
                if (date.getFullYear() > this.disabled.from.getFullYear()) {
                    disabled = true
                }
            }

            return disabled
        },

        /**
         * Set the datepicker value
         * @param {Date|null} date
         */
        setValue(date) {
            if (!date) {
                const d = new Date()
                this.currDate = new Date(d.getFullYear(), d.getMonth(), 1).getTime()
                this.selectedDate = this.formattedValue = null
                return
            }
            this.selectedDate = date
            this.currDate = new Date(date.getFullYear(), date.getMonth(), 1).getTime()
            this.formattedValue = DateUtils.formatDate(date, this.format, this.translation)
        }
    },
    compiled() {
        if (this.value) {
            this.setValue(this.value)
        }
    },
    ready() {
        if (this.isInline()) {
            this.showDayCalendar()
        }
        document.addEventListener('click', (e)=> {
            if (this.$el && !this.$el.contains(e.target)) {
                (this.isInline()) ? this.showDayCalendar() : this.close()
            }
        }, false)
    }
}
</script>

<style lang="stylus">

$width = 300px

*
    box-sizing border-box

.datepicker
    position relative

.calendar
    position absolute
    z-index 100
    background white
    width $width
    border 1px solid #ccc
    header
        display block
        line-height 40px
        span
            display inline-block
            text-align center
            width (100 - (100/7)*2)%
            float left

        .prev
        .next
            width (100/7)%
            float left
            text-indent -10000px
            position relative
            &:after
                content ''
                position absolute
                left 50%
                top 50%
                transform translateX(-50%) translateY(-50%)
                border 6px solid transparent

        .prev
            &:after
                border-right 10px solid #000
                margin-left -5px
            &.disabled:after
                border-right 10px solid #ddd
        .next
            &:after
                border-left 10px solid #000
                margin-left 5px
            &.disabled:after
                border-left 10px solid #ddd

        .prev:not(.disabled)
        .next:not(.disabled)
        .up:not(.disabled)
            cursor pointer
            &:hover
                background #eee

    .disabled
        color #ddd
        cursor default

    .cell
        display inline-block
        padding 0 5px
        width (100/7)%
        height 40px
        line-height 40px
        text-align center
        vertical-align middle
        border 1px solid transparent
        &:not(.blank):not(.disabled).day
        &:not(.blank):not(.disabled).month
        &:not(.blank):not(.disabled).year
            cursor pointer
            &:hover
                border 1px solid #4bd
        &.selected
            background #4bd
            &:hover
                background #4bd
        &.grey
            color #888

            &:hover
                background inherit


        &.day-header
            font-size 75%
            cursor inherit
            &:hover
                background inherit

    .month,
    .year
        width 33.333%


</style>