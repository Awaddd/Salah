<template>
  <h1>Salah Times</h1>
  <h3>{{today}}</h3>

  <div style="display: grid; grid-template-columns: max-content max-content; justify-content: center; grid-gap: 5em; font-size: 1.3em;">
    <ul style="display: grid; justify-items: start;">
      <li v-for="(name, key) in state.prayers" :key="key">{{name}}</li>
    </ul>

    <ul style="display: grid;">
      <li v-for="(name, key) in state.prayerTimes" :key="key">{{name}}</li>
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
      return prayer;
    })
  });

</script>
