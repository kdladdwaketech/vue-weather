<template>
  <div id="app" :class="typeof weather.main != 'undefined' && weather.main.temp > 60 ? 'warm' : ''">

    <main>
      <br>
      <br>
      <h1>Kenyatta's Weather App</h1>
      <br>
      <br>
      <div class="search-box">

        <input type="text" class="search-bar" placeholder="Search..." v-model="query" @keypress="fetchWeather" />
      </div>

      <div class="weather-wrap" v-if="typeof weather.main != 'undefined'">
        <div class="location-box">
          <div class="location">{{ weather.name }}, {{ weather.sys.country }}</div>
          <div class="date">{{ dateBuilder() }}</div>
        </div>

        <div class="weather-box">
          <div class="temp">{{ Math.round(weather.main.temp) }}</div>
          <div class="high-low">High: {{ Math.round(weather.main.temp_max) }}°F | Low: {{
            Math.round(weather.main.temp_min) }}°F</div>
          <div class="weather">{{ weather.weather[0].main }}</div>
          <br>
          <br>
          <button @click="saveLocation">Save Location</button>
        </div>

      </div>
      <div class="saved-locations" v-if="savedLocations.length">
        <h3>Saved Locations</h3>
        <ul>
          <li v-for="(location, index) in savedLocations" :key="index">
            <span @click="viewWeather(location)">{{ location }}</span>
            <br>
            <button @click="removeLocation(index)">Remove</button>
          </li>
        </ul>
      </div>
      <div class="navigation">
        <button @click="currentView = 'hourly'; fetchHourly()">Hourly</button>
        <button @click="currentView = 'fiveDay'; fetchFiveDay()">5 Day Forecast</button>
      </div>

      <div v-if="currentView === 'hourly'" class="hourly-view">
        <canvas id="hourlyChart"></canvas>
      </div>

      <div v-if="currentView === 'fiveDay'" class="five-day-view">
        <table>
          <thead>
            <tr>
              <th>Day</th>
              <th>Condition</th>
              <th>High</th>
              <th>Low</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="day in fiveDayForecast" :key="day.date">
              <td>{{ day.date }}</td>
              <td>{{ day.weather }}</td>
              <td>{{ day.high }}°F</td>
              <td>{{ day.low }}°F</td>
            </tr>
          </tbody>
        </table>
      </div>
    </main>
  </div>
</template>

<script>
import { Chart, registerables } from 'chart.js';
export default {
  name: 'App',
  data() {
    return {
      api_key: 'e91644b7ef9c3e4d023a99d53674ddd3',
      url_base: 'https://api.openweathermap.org/data/2.5/',
      query: '',
      weather: {},
      savedLocations: [],
      currentView: '',
      hourlyData: {},
      fiveDayForecast: [],
    };
  },

  methods: {
    fetchWeather(e) {
      if (e.key == "Enter") {
        fetch(`${this.url_base}weather?q=${this.query}&units=imperial&APPID=${this.api_key}`)
          .then(res => res.json())
          .then(this.setResults);
      }
    },

    setResults(results) {
      console.log(results);
      this.weather = results;
    },

    saveLocation() {
      if (this.weather.name && !this.savedLocations.includes(this.weather.name)) {
        this.savedLocations.push(this.weather.name);
      }
    },

    viewWeather(location) {
      this.query = location;
      fetch(`${this.url_base}weather?q=${location}&units=imperial&APPID=${this.api_key}`)
        .then(res => res.json())
        .then(this.setResults);
    },

    removeLocation(index) {
      this.savedLocations.splice(index, 1);
    },

    fetchHourly() {
      fetch(`${this.url_base}forecast?q=${this.query}&units=imperial&APPID=${this.api_key}`)
        .then(res => res.json())
        .then(data => {
          this.hourlyData = data.list.slice(0, 8).map(hour => ({
            time: hour.dt_txt,
            temp: hour.main.temp,
            rain: hour.rain ? hour.rain["3h"] || 0 : 0,
          }));
          this.renderHourlyChart();
        });
    },

    renderHourlyChart() {
      const ctx = document.getElementById("hourlyChart").getContext("2d");
      Chart.register(...registerables)
      new Chart(ctx, {
        type: "bar",
        data: {
          labels: this.hourlyData.map(hour => hour.time),
          datasets: [
            {
              label: "Temperature (°F)",
              data: this.hourlyData.map(hour => hour.temp),
              backgroundColor: "rgba(75, 192, 192, 0.2)",
              borderColor: "rgba(75, 192, 192, 1)",
              borderWidth: 1,
            },
            {
              label: "Rain (mm)",
              data: this.hourlyData.map(hour => hour.rain),
              backgroundColor: "rgba(54, 162, 235, 0.2)",
              borderColor: "rgba(54, 162, 235, 1)",
              borderWidth: 1,
            },
          ],
        },
        options: {
          scales: {
            y: {
              beginAtZero: true,
            },
          },
        },
      });
    },

    fetchFiveDay() {
      fetch(`${this.url_base}forecast?q=${this.query}&units=imperial&APPID=${this.api_key}`)
        .then(res => res.json())
        .then(data => {
          const dailyData = {};
          data.list.forEach(entry => {
            const date = new Date(entry.dt * 1000).toLocaleDateString();
            if (!dailyData[date]) {
              dailyData[date] = {
                date,
                high: entry.main.temp_max,
                low: entry.main.temp_min,
                weather: entry.weather[0].main,
              };
            } else {
              dailyData[date].high = Math.max(dailyData[date].high, entry.main.temp_max);
              dailyData[date].low = Math.min(dailyData[date].low, entry.main.temp_min);
            }
          });
          this.fiveDayForecast = Object.values(dailyData).slice(0, 5);
        });
    },

    dateBuilder() {
      let d = new Date();
      let months = ["January", "February", "March", "April", "May", "June", "July", "August", "September", "October", "November", "December"];

      let days = ["Sunday", "Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday"];

      let day = days[d.getDay()];
      let date = d.getDate();
      let month = months[d.getMonth()];
      let year = d.getFullYear();

      return `${day} ${date} ${month} ${year}`;
    },
  },
};
</script>

<style>
* {
  margin: 0 auto;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'montserrat', sans-serif;
}

#app {
  background-image: url('./assets/cold-bg.jpg');
  background-size: cover;
  background-position: bottom;
  transition: 0.4s;
}

#app.warm {
  background-image: url('./assets/warm-bg.jpg');
}

h1 {
  text-align: center;
  font-family: 'montserrat', sans-serif;
  color: #321b1b;
  text-shadow: 1px 3px rgba(0, 0, 0, 0.25)
}

main {
  max-width: 1200px;
  min-height: 100vh;
  padding: 25px;
  background-image: linear-gradient(to bottom, rgba(0, 0, 0, 0.25), rgba (0, 0, 0, 0.75))
}

.search-box {
  width: 100%;
  margin-bottom: 30px
}

.search-box .search-bar {
  display: block;
  width: 100%;
  padding: 15px;
  color: #313131;
  font-size: 20px;
  appearance: none;
  border: none;
  outline: none;
  background: none;
  box-shadow: 0px 0px 16px rgba(0, 0, 0, 0.25);

  background-color: rgba(255, 255, 255, 0.5);

  border-radius: 0px 16px 0px 16px;
  transition: 0.4s;
}


.search-box .search-bar:focus {
  box-shadow: 0px 0px 16px rgba(0, 0, 0, 0.25);
  background-color: rgba(255, 255, 255, 0.75);
  border-radius: 16px 0px 16px 0px;
}

.location-box .location {
  color: #fff;
  font-size: 32px;
  font-weight: 500;
  text-align: center;
  text-shadow: 1px 3px rgba(0, 0, 0, 0.25);
}

.location-box .date {
  color: #fff;
  font-size: 20px;
  font-weight: 300;
  text-align: center;
  font-style: italic;
}

.high-low {
  color: #fff;
  font-size: 20px;
  font-weight: 300;
  text-align: center;
  font-style: italic;
}

.weather-box {
  text-align: center;
}

.weather-box .temp {
  display: inline-block;
  padding: 10px 25px;
  color: #fff;
  font-size: 102px;
  font-weight: 900;

  text-shadow: 3px 6px rgba(0, 0, 0, 0.25);
  background-color: rgba(255, 255, 255, 0.25);
  border-radius: 16px;
  margin: 30px 0px;

  box-shadow: 3px 6px rgba(0, 0, 0, 0.25);

}

.weather-box .weather {
  color: #fff;
  font-size: 48px;
  font-weight: 700;
  font-style: italic;
  text-shadow: 3px 6px rgba(0, 0, 0, 0.25);
}

button {
  padding: 10px 10px;
  font-size: 24px;
  cursor: pointer;
  text-align: center;
  text-decoration: none;
  outline: none;
  color: #fff;
  background-color: rgba(0, 0, 0, 0.25);
  border: none;
  border-radius: 20px;

}

.saved-locations {
  margin-top: 30px;
  background: rgba(0, 0, 0, 0.25);
  padding: 10px 25px;
  color: #fff;
  font-size: 30px;
  font-weight: 900;
  text-align: center;
}

ul {
  list-style-type: none;
}

li {
  font-size: 25px;
}
</style>
