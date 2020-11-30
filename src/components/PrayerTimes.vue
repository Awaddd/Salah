<template>
  <h1>Salah Times</h1>
  <h3>{{today}}</h3>

  <div style="display: grid; grid-template-columns: max-content max-content; justify-content: center; grid-gap: 5em; font-size: 1.3em;">
    <ul style="display: grid; justify-items: start;">
      <li v-for="(prayerName, key) in state.prayers" :key="key">{{prayerName}}</li>
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
import { ref, reactive, toRefs, computed } from "vue";
import dayjs from "dayjs";
import convertTime from "convert-time";

const state = reactive({
  prayerTimes: {}, 
  prayers: []
});

let today = ref();
today.value = dayjs().format("YYYY-MM-DD");

function previousDate () {
  if (dayjs(today.value).date() > 1) {
    today.value = dayjs(today.value).subtract(1, "day").format("YYYY-MM-DD");
  }
}

function nextDate () {
  if (dayjs(today.value).date() < dayjs(today.value).daysInMonth()) {
    today.value = dayjs(today.value).add(1, "day").format("YYYY-MM-DD");
  }
}

fetch(`https://www.londonprayertimes.com/api/times/?format=json&key=${apiKey}&year=${dayjs().year()}&month=${dayjs().month() + 1}`)
  .then(response => response.json())
  .then(data => {
    state.prayers = ['fajr', 'sunrise', 'dhuhr', 'asr', 'magrib', 'isha'];

    state.prayerTimes = computed(() => {
      let prayer = {};

      for (let prayerTime in data.times[today.value]) {
        if (state.prayers.includes(prayerTime)) {
          prayer[prayerTime] = data.times[today.value][prayerTime]
        }
      }

      prayer.fajr = convertTime(prayer.fajr + 'am');
      prayer.sunrise = convertTime(prayer.sunrise + 'am');

      let dhuhrLongDate = dayjs(today.value + ' ' + prayer.dhuhr);
      prayer.dhuhr = dhuhrLongDate.isBefore(today.value + ' 11:59') ? convertTime(prayer.dhuhr + 'am') : convertTime(prayer.dhuhr + 'pm');

      prayer.asr = convertTime(prayer.asr + 'pm');
      prayer.magrib = convertTime(prayer.magrib + 'pm');
      prayer.isha = convertTime(prayer.isha + 'pm');

      return prayer;
    });   
  });

</script>
