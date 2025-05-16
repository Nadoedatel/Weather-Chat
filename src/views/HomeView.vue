<template>
  <div class="weather-app" :class="weatherClass">
    <h1>Погодное приложение</h1>
    <div class="search-box">
      <input 
        v-model="cityQuery" 
        placeholder="Введите город" 
        @keyup.enter="fetchWeather"
      />
      <button @click="fetchWeather">Поиск</button>
    </div>

    <div v-if="loading" class="loading">Загрузка...</div>

    <div v-if="error" class="error">{{ error }}</div>

    <div v-if="weatherData" class="weather-card">
      <h2>{{ weatherData.cityName }}, {{ weatherData.country }}</h2>
      <div class="weather-main">
        <img 
          :src="`https://openweathermap.org/img/wn/${weatherData.icon}@2x.png`" 
          :alt="weatherData.description"
        />
        <span class="temp">{{ Math.round(weatherData.temp) }}°C</span>
      </div>
      <p class="description">{{ weatherData.description }}</p>
      <div class="weather-details">
        <p>Ощущается как: {{ Math.round(weatherData.feels_like) }}°C</p>
        <p>Влажность: {{ weatherData.humidity }}%</p>
        <p>Ветер: {{ Math.round(weatherData.wind_speed) }} м/с</p>
        <p>Давление: {{ weatherData.pressure }} гПа</p>
      </div>
    </div>
    <div v-if="showRain" ref="rainContainer" class="rain-container"></div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue';
import axios from 'axios';
import * as RainJS from 'rain.js';

const API_KEY = 'd323032a8c6a1e5005c7097d36436159';
const cityQuery = ref('Москва');
const weatherData = ref(null);
const loading = ref(false);
const error = ref(null);
const rainContainer = ref(null);
let rainEffect = null;

const showRain = computed(() => {
  const desc = weatherData.value?.description.toLowerCase() || '';
  return desc.includes('дождь') || desc.includes('rain');
});

const weatherClass = computed(() => {
  return showRain.value ? 'rainy-theme' : 'default-theme';
});

onMounted(() => {
  fetchWeather();
});

onUnmounted(() => {
  if (rainEffect) {
    rainEffect.stop();
    rainEffect = null;
  }
});

const initRainEffect = () => {
  if (!rainContainer.value) return;
  
  if (rainEffect) {
    rainEffect.stop();
  }

  rainEffect = new Rain({
    target: rainContainer.value,
    speed: 1,
    density: 0.5,
    angle: 5,
    color: '#a0c4e6'
  });
  
  rainEffect.start();
};

const fetchWeather = async () => {
  if (!cityQuery.value.trim()) return;
  
  loading.value = true;
  error.value = null;
  weatherData.value = null;

  try {
    const geoResponse = await axios.get(
      `https://api.openweathermap.org/geo/1.0/direct?q=${cityQuery.value}&limit=1&appid=${API_KEY}`
    );

    if (!geoResponse.data?.length) {
      throw new Error('Город не найден');
    }

    const { lat, lon, name: cityName, country } = geoResponse.data[0];
    const weatherResponse = await axios.get(
      `https://api.openweathermap.org/data/2.5/weather?lat=${lat}&lon=${lon}&units=metric&lang=ru&appid=${API_KEY}`
    );

    weatherData.value = {
      cityName,
      country,
      temp: weatherResponse.data.main.temp,
      feels_like: weatherResponse.data.main.feels_like,
      humidity: weatherResponse.data.main.humidity,
      pressure: weatherResponse.data.main.pressure,
      wind_speed: weatherResponse.data.wind.speed,
      description: weatherResponse.data.weather[0].description,
      icon: weatherResponse.data.weather[0].icon
    };

    // Инициализируем эффект дождя после получения данных
    if (showRain.value) {
      setTimeout(initRainEffect, 0);
    }
  } catch (err) {
    error.value = err.response?.data?.message || err.message || 'Произошла ошибка';
    console.error('Ошибка при получении погоды:', err);
  } finally {
    loading.value = false;
  }
};
</script>

<style>
.weather-app {
  max-width: 500px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  text-align: center;
  min-height: 100vh;
  position: relative;
  transition: background 0.5s ease;
}

.default-theme {
  background: #4CAF50;
  color: black;
}

.rainy-theme {
  background: linear-gradient(to bottom, #4a6b8a, #2c3e50);
  color: #ecf0f1;
}

.search-box {
  margin: 20px 0;
}

input {
  padding: 10px;
  width: 200px;
  margin-right: 10px;
  background: white;
  color: black;
  border: none;
  border-radius: 4px;
}

button {
  padding: 10px 15px;
  background-color: #2c3e50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:hover {
  background-color: #1a252f;
}

.weather-card {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 8px;
  padding: 20px;
  margin-top: 20px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
  backdrop-filter: blur(5px);
}

.weather-main {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 15px 0;
}

.temp {
  font-size: 2.5rem;
  font-weight: bold;
  margin-left: 10px;
}

.description {
  text-transform: capitalize;
  font-size: 1.2rem;
  margin-bottom: 15px;
}

.weather-details {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 10px;
  text-align: left;
  margin-top: 15px;
}

.loading {
  margin: 20px 0;
  color: inherit;
}

.error {
  color: #ff6b6b;
  margin: 20px 0;
}

.rain-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  pointer-events: none;
  z-index: 100;
  opacity: 0.7;
}
</style>