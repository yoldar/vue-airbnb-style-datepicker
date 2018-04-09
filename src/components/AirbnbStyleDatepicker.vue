<template>
  <transition name="fade">
    <div
      :id="wrapperId"
      class="airbnb-style-datepicker-wrapper"
      v-show="showDatepicker"
      :class="wrapperClasses"
      :style="showFullscreen ? undefined : wrapperStyles"
      v-click-outside="handleClickOutside"
    >

      <div class="datepicker-header">
        <div class="change-month-button previous">
          <button @click="previousMonth">
            <svg viewBox="0 0 1000 1000"><path d="M336.2 274.5l-210.1 210h805.4c13 0 23 10 23 23s-10 23-23 23H126.1l210.1 210.1c11 11 11 21 0 32-5 5-10 7-16 7s-11-2-16-7l-249.1-249c-11-11-11-21 0-32l249.1-249.1c21-21.1 53 10.9 32 32z" /></svg>
          </button>
        </div>
        <div class="change-month-button next">
          <button @click="nextMonth">
            <svg viewBox="0 0 1000 1000"><path d="M694.4 242.4l249.1 249.1c11 11 11 21 0 32L694.4 772.7c-5 5-10 7-16 7s-11-2-16-7c-11-11-11-21 0-32l210.1-210.1H67.1c-13 0-23-10-23-23s10-23 23-23h805.4L662.4 274.5c-21-21.1 11-53.1 32-32.1z" /></svg>
          </button>
        </div>

        <div
          class="days-legend"
          v-for="(month, index) in showMonths"
          :key="month"
          :style="[monthWidthStyles, {left: (width * index) + 'px'}]"
        >
          <div class="day-title" v-for="day in daysShort" :key="day">{{ day }}</div>
        </div>
      </div>

      <div class="datepicker-inner-wrapper" :style="innerStyles">
        <transition-group name="list-complete" tag="div">
          <div
            v-for="(month, monthIndex) in months"
            :key="month.firstDateOfMonth"
            class="month"
            :class="{hidden: monthIndex === 0 || monthIndex > showMonths}"
            :style="monthWidthStyles"
          >
            <div class="month-name">{{ monthNames[month.monthIndex] }} {{ month.year }}</div>

            <table class="month-table" role="presentation">
              <tbody>
                <tr class="week" v-for="(week, index) in month.weeks" :key="index">
                  <td
                    class="day"
                    v-for="({fullDate, dayNumber}, index) in week"
                    :key="index + '_' + dayNumber"
                    :data-date="fullDate"
                    :class="{
                      enabled: dayNumber !== 0,
                      empty: dayNumber === 0,
                      disabled: isDisabled(fullDate),
                      selected: selectedDate1 === fullDate || selectedDate2 === fullDate,
                      'in-range': isInRange(fullDate)
                    }"
                    :style="{
                      width: (width - 30) / 7 + 'px',
                      background: isSelected(fullDate) ? colors.selected : isInRange(fullDate) ? colors.inRange : '',
                      color: isSelected(fullDate) ? colors.selectedText : isInRange(fullDate) ? colors.selectedText : colors.text
                    }"
                    @mouseover="() => { setHoverDate(fullDate) }"
                  >
                    <button
                      class="day-button"
                      v-if="dayNumber"
                      :date="fullDate"
                      :disabled="isDisabled(fullDate)"
                      @click="() => { selectDate(fullDate) }"
                    >{{ dayNumber }}</button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </transition-group>
      </div>
      <div class="action-buttons mobile-only" v-if="mode !== 'single' && showActionButtons">
        <button @click="closeDatepickerCancel">{{ texts.cancel }}</button>
        <button @click="closeDatepicker" :style="{color: colors.selected}">{{ texts.apply }}</button>
      </div>
    </div>
  </transition>
</template>

<script>
import format from 'date-fns/format'
import subMonths from 'date-fns/sub_months'
import addMonths from 'date-fns/add_months'
import getDaysInMonth from 'date-fns/get_days_in_month'
import isBefore from 'date-fns/is_before'
import isAfter from 'date-fns/is_after'
import isValid from 'date-fns/is_valid'
import Languages from './languages.js'
import { debounce, copyObject, findAncestor, randomString } from './../helpers'

export default {
  name: 'AirbnbStyleDatepicker',
  props: {
    triggerElementId: { type: String },
    dateOne: { type: [String, Date], default: format(new Date()) },
    dateTwo: { type: [String, Date] },
    minDate: { type: [String, Date] },
    endDate: { type: [String, Date] },
    mode: { type: String, default: 'range' },
    offsetY: { type: Number, default: 0 },
    offsetX: { type: Number, default: 0 },
    monthsToShow: { type: Number, default: 1 },
    startOpen: { type: Boolean },
    fullscreenMobile: { type: Boolean, default: true },
    inline: { type: Boolean },
    disabledDates: { type: Array, default: () => [] },
    showActionButtons: { type: Boolean, default: true },
    disabled: { type: Boolean, default: false },
    isTest: {
      type: Boolean,
      default: () => process.env.NODE_ENV === 'test'
    },
    lang: { type: String, default: 'ru' }
  },
  data() {
    return {
      wrapperId: 'airbnb-style-datepicker-wrapper-' + randomString(5),
      dateFormat: 'YYYY-MM-DD',
      showDatepicker: false,
      showMonths: 2,
      colors: {
        selected: '#643184',
        inRange: '#b98ad1',
        selectedText: '#fff',
        text: '#73879c'
      },
      sundayFirst: false,
      startingDate: '',
      months: [],
      width: 300,
      selectedDate1: '',
      selectedDate2: '',
      isSelectingDate1: true,
      hoverDate: '',
      alignRight: false,
      triggerPosition: {},
      triggerWrapperPosition: {},
      viewportWidth: window.innerWidth + 'px',
      isMobile: window.innerWidth < 768,
      isTablet: window.innerWidth >= 768 && window.innerWidth <= 1024,
      triggerElement: undefined
    }
  },
  computed: {
    translation() {
      return Languages[this.lang]
    },
    daysShort() {
      return this.translation.daysShort
    },
    monthNames() {
      return this.translation.monthNames
    },
    texts() {
      return this.translation.texts
    },
    wrapperClasses() {
      return {
        'datepicker-open': this.showDatepicker,
        'full-screen': this.showFullscreen,
        inline: this.inline
      }
    },
    wrapperStyles() {
      return {
        position: this.inline ? 'static' : 'absolute',
        top: this.inline
          ? '0'
          : this.triggerPosition.height + this.offsetY + 'px',
        left: !this.alignRight
          ? this.triggerPosition.left -
            this.triggerWrapperPosition.left +
            this.offsetX +
            'px'
          : '',
        right: this.alignRight
          ? this.triggerWrapperPosition.right -
            this.triggerPosition.right +
            this.offsetX +
            'px'
          : '',
        width: this.width * this.showMonths + 'px',
        zIndex: this.inline ? '0' : '100'
      }
    },
    innerStyles() {
      return {
        'margin-left': this.showFullscreen
          ? '-' + this.viewportWidth
          : `-${this.width}px`
      }
    },
    monthWidthStyles() {
      return {
        width: this.showFullscreen ? this.viewportWidth : this.width + 'px'
      }
    },
    showFullscreen() {
      return this.isMobile && this.fullscreenMobile
    },
    datesSelected() {
      return !!(
        (this.selectedDate1 && this.selectedDate1 !== '') ||
        (this.selectedDate2 && this.selectedDate2 !== '')
      )
    },
    allDatesSelected() {
      return !!(
        this.selectedDate1 &&
        this.selectedDate1 !== '' &&
        this.selectedDate2 &&
        this.selectedDate2 !== ''
      )
    },
    hasMinDate() {
      return !!(this.minDate && this.minDate !== '')
    },
    isRangeMode() {
      return this.mode === 'range'
    },
    isSingleMode() {
      return this.mode === 'single'
    },
    datepickerWidth() {
      return this.width * this.showMonths
    },
    datePropsCompound() {
      // used to watch for changes in props, and update GUI accordingly
      return this.dateOne + this.dateTwo
    },
    isDateTwoBeforeDateOne() {
      if (!this.dateTwo) {
        return false
      }
      return isBefore(this.dateTwo, this.dateOne)
    }
  },
  watch: {
    selectedDate1(newValue, oldValue) {
      let newDate =
        !newValue || newValue === '' ? '' : format(newValue, this.dateFormat)
      this.$emit('date-one-selected', newDate)
    },
    selectedDate2(newValue, oldValue) {
      let newDate =
        !newValue || newValue === '' ? '' : format(newValue, this.dateFormat)
      this.$emit('date-two-selected', newDate)
    },
    mode(newValue, oldValue) {
      this.setStartDates()
    },
    datePropsCompound(newValue) {
      if (this.dateOne !== this.selectedDate1) {
        this.startingDate = this.dateOne
        this.setStartDates()
        this.generateMonths()
      }
      if (this.isDateTwoBeforeDateOne) {
        this.selectedDate2 = ''
        this.$emit('date-two-selected', '')
      }
    }
  },
  created() {
    this.setupDatepicker()

    if (this.sundayFirst) {
      this.setSundayToFirstDayInWeek()
    }

    this._handleWindowResizeEvent = debounce(() => {
      this.positionDatepicker()
      this.setStartDates()
    }, 200)
    this._handleWindowClickEvent = event => {
      if (event.target.id === this.triggerElementId) {
        event.stopPropagation()
        event.preventDefault()
        this.toggleDatepicker()
      }
    }
    window.addEventListener('resize', this._handleWindowResizeEvent)
    window.addEventListener('click', this._handleWindowClickEvent)
  },
  mounted() {
    this.triggerElement = this.isTest
      ? document.createElement('input')
      : document.getElementById(this.triggerElementId)

    this.setStartDates()
    this.generateMonths()

    if (this.startOpen || this.inline) {
      this.openDatepicker()
    }

    this.triggerElement.addEventListener('keyup', this.handleTriggerInput)
  },
  destroyed() {
    window.removeEventListener('resize', this._handleWindowResizeEvent)
    window.removeEventListener('click', this._handleWindowClickEvent)

    this.triggerElement.removeEventListener('keyup', this.handleTriggerInput)
  },
  methods: {
    handleClickOutside(event) {
      if (event.target.id === this.triggerElementId) {
        return
      }
      this.closeDatepicker()
    },
    handleTriggerInput(event) {
      const keys = {
        arrowDown: 40,
        arrowUp: 38,
        arrowRight: 39,
        arrowLeft: 37
      }
      if (
        event.keyCode === keys.arrowDown &&
        !event.shiftKey &&
        !this.showDatepicker
      ) {
        this.openDatepicker()
      } else if (
        event.keyCode === keys.arrowUp &&
        !event.shiftKey &&
        this.showDatepicker
      ) {
        this.closeDatepicker()
      } else if (
        event.keyCode === keys.arrowRight &&
        !event.shiftKey &&
        this.showDatepicker
      ) {
        this.nextMonth()
      } else if (
        event.keyCode === keys.arrowLeft &&
        !event.shiftKey &&
        this.showDatepicker
      ) {
        this.previousMonth()
      } else {
        if (this.mode === 'single') {
          this.setDateFromText(event.target.value)
        }
      }
    },
    setDateFromText(value) {
      if (value.length < 10) {
        return
      }
      // make sure format is either 'YYYY-MM-DD' or 'DD.MM.YYYY'
      const isFormatYearFirst = value.match(
        /^(\d{4})-(0[1-9]|1[0-2])-(0[1-9]|1[0-9]|2[0-9]|3[0-1])$/
      )
      const isFormatDayFirst = value.match(
        /^(0[1-9]|1[0-9]|2[0-9]|3[0-1])[.](0[1-9]|1[0-2])[.](\d{4})$/
      )

      if (!isFormatYearFirst && !isFormatDayFirst) {
        return
      }
      if (isFormatDayFirst) {
        //convert to YYYY-MM-DD
        value = `${value.substring(6, 10)}-${value.substring(
          3,
          5
        )}-${value.substring(0, 2)}`
      }

      const valueAsDateObject = new Date(value)
      if (!isValid(valueAsDateObject)) {
        return
      }
      const formattedDate = format(valueAsDateObject, this.dateFormat)
      if (
        this.isDateDisabled(formattedDate) ||
        this.isBeforeMinDate(formattedDate) ||
        this.isAfterEndDate(formattedDate)
      ) {
        return
      }
      this.startingDate = subMonths(formattedDate, 1)
      this.generateMonths()
      this.selectDate(formattedDate)
    },
    generateMonths() {
      this.months = []
      for (let i = 0; i < this.showMonths + 2; i++) {
        this.months.push(this.getMonth(this.startingDate))
        this.startingDate = this.addMonths(this.startingDate)
      }
    },
    setupDatepicker() {
      if (this.$options.sundayFirst) {
        this.sundayFirst = copyObject(this.$options.sundayFirst)
      }
      if (this.$options.colors) {
        const colors = copyObject(this.$options.colors)
        this.colors.selected = colors.selected || this.colors.selected
        this.colors.inRange = colors.inRange || this.colors.inRange
        this.colors.selectedText =
          colors.selectedText || this.colors.selectedText
        this.colors.text = colors.text || this.colors.text
      }
    },
    setStartDates() {
      let startDate = this.dateOne || new Date()
      if (this.hasMinDate && isBefore(startDate, this.minDate)) {
        startDate = this.minDate
      }
      this.startingDate = this.subtractMonths(startDate)
      this.selectedDate1 = this.dateOne
      this.selectedDate2 = this.dateTwo
    },
    setSundayToFirstDayInWeek() {
      const lastDayShort = this.daysShort.pop()
      this.daysShort.unshift(lastDayShort)
    },
    getMonth(date) {
      const firstDateOfMonth = format(date, 'YYYY-MM-01')
      const year = format(date, 'YYYY')
      const monthNumber = parseInt(format(date, 'M'))
      const monthIndex = monthNumber - 1

      return {
        year,
        firstDateOfMonth,
        monthIndex,
        monthNumber,
        weeks: this.getWeeks(firstDateOfMonth)
      }
    },
    getWeeks(date) {
      const weekDayNotInMonth = { dayNumber: 0 }
      const daysInMonth = getDaysInMonth(date)
      const year = format(date, 'YYYY')
      const month = format(date, 'MM')
      let firstDayInWeek = parseInt(format(date, this.sundayFirst ? 'd' : 'E'))
      if (this.sundayFirst) {
        firstDayInWeek++
      }
      let weeks = []
      let week = []

      // add empty days to get first day in correct position
      for (let s = 1; s < firstDayInWeek; s++) {
        week.push(weekDayNotInMonth)
      }
      for (let d = 0; d < daysInMonth; d++) {
        let isLastDayInMonth = d >= daysInMonth - 1
        let dayNumber = d + 1
        let dayNumberFull = dayNumber < 10 ? '0' + dayNumber : dayNumber
        week.push({
          dayNumber,
          dayNumberFull: dayNumberFull,
          fullDate: year + '-' + month + '-' + dayNumberFull
        })

        if (week.length === 7) {
          weeks.push(week)
          week = []
        } else if (isLastDayInMonth) {
          for (let i = 0; i < 7 - week.length; i++) {
            week.push(weekDayNotInMonth)
          }
          weeks.push(week)
          week = []
        }
      }
      return weeks
    },
    selectDate(date) {
      if (
        this.isBeforeMinDate(date) ||
        this.isAfterEndDate(date) ||
        this.isDateDisabled(date)
      ) {
        return
      }

      if (this.mode === 'single') {
        this.selectedDate1 = date
        this.closeDatepicker()
        return
      }

      if (this.isSelectingDate1 || isBefore(date, this.selectedDate1)) {
        this.selectedDate1 = date
        this.isSelectingDate1 = false

        if (isBefore(this.selectedDate2, date)) {
          this.selectedDate2 = ''
        }
      } else {
        this.selectedDate2 = date
        this.isSelectingDate1 = true

        if (isAfter(this.selectedDate1, date)) {
          this.selectedDate1 = ''
        }
      }
    },
    setHoverDate(date) {
      this.hoverDate = date
    },
    isSelected(date) {
      if (!date) {
        return
      }
      return this.selectedDate1 === date || this.selectedDate2 === date
    },
    isInRange(date) {
      if (!this.allDatesSelected || this.isSingleMode) {
        return false
      }

      return (
        (isAfter(date, this.selectedDate1) &&
          isBefore(date, this.selectedDate2)) ||
        (isAfter(date, this.selectedDate1) &&
          isBefore(date, this.hoverDate) &&
          !this.allDatesSelected)
      )
    },
    isBeforeMinDate(date) {
      if (!this.minDate) {
        return false
      }
      return isBefore(date, this.minDate)
    },
    isAfterEndDate(date) {
      if (!this.endDate) {
        return false
      }
      return isAfter(date, this.endDate)
    },
    isDateDisabled(date) {
      const isDisabled = this.disabledDates.indexOf(date) > -1
      return isDisabled
    },
    isDisabled(date) {
      return (
        this.isDateDisabled(date) ||
        this.isBeforeMinDate(date) ||
        this.isAfterEndDate(date)
      )
    },
    previousMonth() {
      this.startingDate = this.subtractMonths(this.months[0].firstDateOfMonth)

      this.months.unshift(this.getMonth(this.startingDate))
      this.months.splice(this.months.length - 1, 1)
    },
    nextMonth() {
      this.startingDate = this.addMonths(
        this.months[this.months.length - 1].firstDateOfMonth
      )
      this.months.push(this.getMonth(this.startingDate))

      setTimeout(() => {
        this.months.splice(0, 1)
      }, 100)
    },
    subtractMonths(date) {
      return format(subMonths(date, 1), this.dateFormat)
    },
    addMonths(date) {
      return format(addMonths(date, 1), this.dateFormat)
    },
    toggleDatepicker() {
      if (this.showDatepicker) {
        this.closeDatepicker()
      } else if (!this.disabled) {
        this.openDatepicker()
      }
    },
    openDatepicker() {
      this.positionDatepicker()
      this.setStartDates()
      this.triggerElement.classList.add('datepicker-open')
      this.showDatepicker = true
      this.initialDate1 = this.dateOne
      this.initialDate2 = this.dateTwo
    },
    closeDatepickerCancel() {
      if (this.showDatepicker) {
        this.selectedDate1 = this.initialDate1
        this.selectedDate2 = this.initialDate2
        this.closeDatepicker()
      }
    },
    closeDatepicker() {
      if (this.inline) {
        return
      }
      this.showDatepicker = false
      this.triggerElement.classList.remove('datepicker-open')
      this.$emit('closed')
    },
    positionDatepicker() {
      const triggerWrapperElement = findAncestor(
        this.triggerElement,
        '.datepicker-trigger'
      )
      this.triggerPosition = this.triggerElement.getBoundingClientRect()
      if (triggerWrapperElement) {
        this.triggerWrapperPosition = triggerWrapperElement.getBoundingClientRect()
      } else {
        this.triggerWrapperPosition = { left: 0, right: 0 }
      }

      const viewPortWidth = Math.max(
        document.documentElement.clientWidth,
        window.innerWidth || 0
      )
      this.viewportWidth = viewPortWidth + 'px'
      this.isMobile = viewPortWidth < 768
      this.isTablet = viewPortWidth >= 768 && viewPortWidth <= 1024
      this.showMonths = this.isMobile
        ? 1
        : this.isTablet && this.monthsToShow > 2 ? 2 : this.monthsToShow

      this.$nextTick(function() {
        const datepickerWrapper = document.getElementById(this.wrapperId)
        if (!this.triggerElement || !datepickerWrapper) {
          return
        }

        const rightPosition =
          this.triggerElement.getBoundingClientRect().left +
          datepickerWrapper.getBoundingClientRect().width
        this.alignRight = rightPosition > viewPortWidth
      })
    }
  }
}
</script>

<style lang="scss">
@import './../styles/transitions';

$tablet: 768px;
$color-gray: rgba(0, 0, 0, 0.2);
$transition-time: 0.3s;
$border-normal: 1px solid $color-gray;

*,
*:after,
*:before {
  box-sizing: border-box;
}

.datepicker-trigger {
  position: relative;
  overflow: visible;
}

.airbnb-style-datepicker-wrapper {
  border: $border-normal;
  white-space: nowrap;
  text-align: center;
  overflow: hidden;
  background-color: white;
  border: 1px solid #d9d9d9;
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.175);

  &.full-screen {
    position: fixed;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
    border: none;
    z-index: 100;
  }
  .datepicker-inner-wrapper {
    transition: all $transition-time ease;
    position: relative;
  }
  .datepicker-header {
    position: relative;
  }
  .change-month-button {
    position: absolute;
    top: 12px;
    z-index: 10;
    background: white;

    &.previous {
      left: 0;
      padding-left: 15px;
    }
    &.next {
      right: 0;
      padding-right: 15px;
    }

    button {
      background-color: white;
      border: none;
      padding: 4px 8px;
      cursor: pointer;
    }

    svg {
      height: 19px;
      width: 19px;
      fill: #82888a;
    }
  }
  .days-legend {
    position: absolute;
    top: 50px;
    left: 10px;
    padding: 0 10px;
  }
  .day-title {
    display: inline-block;
    width: percentage(1/7);
    text-align: center;
    margin-bottom: 4px;
    color: rgba(0, 0, 0, 0.7);
    font-size: 0.8em;
    margin-left: -1px;
    font-weight: bold;
  }

  .month-table {
    border-collapse: collapse;
    border-spacing: 0;
    background: white;
    width: 100%;
    max-width: 100%;
  }

  .month {
    transition: all $transition-time ease;
    display: inline-block;
    padding: 15px;

    &.hidden {
      height: auto;
      visibility: hidden;
    }
  }
  .month-name {
    font-size: 1.3em;
    text-align: center;
    margin: 0 0 30px;
    line-height: 1.4em;
    font-weight: normal;
  }

  .day {
    $size: 26px;
    line-height: normal;
    height: $size;
    padding: 0;
    overflow: hidden;

    &:not(.disabled):hover {
      background-color: #e4e7e7;
    }

    &.selected,
    &.in-range {
    }
    &.disabled,
    &.empty {
      opacity: 0.2;

      button {
        cursor: default;
      }
    }
    &.empty {
      border: none;
    }
    &.empty:hover {
      background-color: #fff;
    }
  }
  .day-button {
    background: transparent;
    width: 100%;
    height: 100%;
    border: none;
    cursor: pointer;
    color: inherit;
    text-align: center;
    user-select: none;
    font-size: 15px;
    font-weight: inherit;
    padding: 0;
  }

  .action-buttons {
    min-height: 50px;
    padding-top: 10px;
    button {
      display: block;
      position: relative;
      background: transparent;
      border: none;
      font-weight: normal;
      font-size: 14px;
      cursor: pointer;

      &:hover {
        text-decoration: underline;
      }
      &:nth-child(1) {
        float: left;
        left: 15px;
      }
      &:nth-child(2) {
        float: right;
        right: 15px;
      }
    }
  }

  .mobile-only {
    display: none;
    @media (max-width: 767px) {
      display: block;
    }
  }
}
</style>
