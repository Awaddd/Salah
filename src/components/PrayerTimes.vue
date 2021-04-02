<template>
  <h3 style="text-transform: capitalize;">{{state.currentPrayer}}</h3>
  <h2 style="text-transform: capitalize;">{{state.currentPrayerTime}}</h2>
  <h1 style="margin-top: 0; height: 2.4rem;"><span v-if="state.countdown != null">-{{state.countdown}}</span></h1>
  <h1 style="margin-top: 0;">{{state.today}}</h1>

  <div style="display: grid; grid-template-columns: max-content max-content; justify-content: center; grid-gap: 5em; font-size: 1.3em;">
    <ul style="display: grid; justify-items: start;">
      <li v-for="(prayerName, key) in state.prayerTimeKeys" :key="key">{{prayerName}}</li>
    </ul>

    <ul style="display: grid;">
      <li v-for="(prayerTime, key) in state.prayerTimes" :key="key">{{prayerTime}}</li>
    </ul>
  </div>
  
  <div class="controls" style="display: grid; grid-gap: 0.5em; grid-template-columns: min-content min-content; justify-content: center;">
    <button @click="previousDate">previous</button>
    <button @click="nextDate">next</button>
  </div>

</template>

<script setup>

import { apiKey } from '../../config.json';
import { reactive, computed, watchEffect } from "vue";
import dayjs from "dayjs";
import isBetween from '../../node_modules/dayjs/esm/plugin/isBetween/index.js'
import advancedFormat from '../../node_modules/dayjs/esm/plugin/advancedFormat/index.js'
import convertTime from "convert-time";
import Timer from 'tiny-timer';

dayjs.extend(isBetween);
dayjs.extend(advancedFormat);

const state = reactive({
  prayerTimes: {},
  prayerTimeKeys: [],
  today: dayjs().format("YYYY-MM-DD"),
  userFriendlyDateToday: dayjs().format("Do MMMM YYYY"),
  now: dayjs().format('HH:mm'),
  countdown: null
});

let prayerContainer = [];

setInterval(() => {
  state.now = dayjs().format('HH:mm');
}, 1000);

function previousDate () {
  if (dayjs(state.today).date() > 1) {
    state.today = dayjs(state.today).subtract(1, "day").format("YYYY-MM-DD");
    state.userFriendlyDateToday = dayjs(state.today).format("Do MMMM YYYY");
  }
}

function nextDate () {
  if (dayjs(state.today).date() < dayjs(state.today).daysInMonth()) {
    state.today = dayjs(state.today).add(1, "day").format("YYYY-MM-DD");
    state.userFriendlyDateToday = dayjs(state.today).format("Do MMMM YYYY");
  }
}

function getNextPrayer () {
  Object.keys(prayerContainer).reduce((previous, key) => {

  // previous = value from previous iteration
  // prayerContainer[key].time = value from current iteration

  // if initial evaluation, skip as previous and current will be the same
  if (previous == prayerContainer[key].time) {
    return prayerContainer[key].time;
  }

  if (dayjs(`${state.today} ${state.now}`).isBetween(`${state.today} ${prayerContainer[key].time}`, dayjs(`${state.today} ${previous}`)) == true) {
    state.currentPrayer = prayerContainer[key].name;
    state.currentPrayerTime = prayerContainer[key].time;
  }

  return prayerContainer[key].time;
  }, prayerContainer[0].time);

  // if currentPrayer is still not defined, set the current prayer to the last prayer
  if (typeof state.currentPrayer == 'undefined') state.currentPrayer = 'isha';
}

fetch(`https://www.londonprayertimes.com/api/times/?format=json&key=${apiKey}&year=${dayjs().year()}&month=${dayjs().month() + 1}`)
  .then(response => response.json())
  .then(data => {
    state.prayerTimeKeys = ['fajr', 'sunrise', 'dhuhr', 'asr', 'magrib', 'isha'];

    state.prayerTimes = computed(() => {
      let prayer = {};

      for (let prayerTime in data.times[state.today]) {
        if (state.prayerTimeKeys.includes(prayerTime)) {
          prayer[prayerTime] = data.times[state.today][prayerTime]
        }
      }

      prayer.fajr = convertTime(prayer.fajr + 'am');
      prayer.sunrise = convertTime(prayer.sunrise + 'am');

      let dhuhrLongDate = dayjs(state.today + ' ' + prayer.dhuhr);
      prayer.dhuhr = dhuhrLongDate.isAfter(state.today + ' 11:00') ? convertTime(prayer.dhuhr + 'am') : convertTime(prayer.dhuhr + 'pm');

      prayer.asr = convertTime(prayer.asr + 'pm');
      prayer.magrib = convertTime(prayer.magrib + 'pm');
      prayer.isha = convertTime(prayer.isha + 'pm');

      let i = 0;
      prayerContainer[i] = {
        name: 'midnight',
        time: '00:00'
      }
      i++;

      for (let prayerName in prayer) {
        prayerContainer[i] = {
          name: prayerName,
          time: prayer[prayerName]
        };
        i++;
      }
      
      return prayer;
    });  
  })
  .then(() => {
    getNextPrayer();
  });

  watchEffect(() => {
    if (dayjs(`${state.today} ${state.now}`).isAfter(dayjs(`${state.today} 23:59`))) {
      state.currentPrayer = 'fajr';
    }

    if (state.currentPrayerTime != null) {
      // initialize timer if the next prayer/time is set
      startTimer()
    }
  });

  function startTimer() {
    // get time until next prayer in milliseconds and pass that into the timer
    let timeTillNextEvent = dayjs(`${state.today} ${state.currentPrayerTime}`).diff(dayjs(`${state.today} ${state.now}`));
    timer.start(timeTillNextEvent);
    getNextPrayer();
  }

  const timer = new Timer();

  timer.on('done', () => {
    // once next prayer time is reached, restart timer
    startTimer()
  });
  
  timer.on('tick', (ms) => {
    // update the countdown every millisecond
    state.countdown = dayjs(ms).subtract(1, "hour").format('HH:mm:ss')
  })

</script>
