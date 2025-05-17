<template>
  <div class="weather-app" :class="weatherTheme">
    <h1>Погодное приложение</h1>
    <div class="search-box">
      <input
        v-model="cityQuery"
        placeholder="Введите город"
        @keyup.enter="fetchWeather"
        :disabled="loading"
      />
      <button @click="fetchWeather" :disabled="loading || !cityQuery.trim()">
        {{ loading ? 'Загрузка...' : 'Поиск' }}
      </button>
    </div>

    <div v-if="loading" class="loading">
      <div class="spinner"></div>
      <p>Загрузка данных...</p>
    </div>

    <div v-if="error" class="error">
      <p>{{ error }}</p>
      <button @click="retryFetch">Попробовать снова</button>
    </div>

    <div v-if="weatherData" class="weather-card">
      <h2>{{ weatherData.cityName }}, {{ weatherData.country }}</h2>
      <div class="weather-main">
        <img
          :src="`https://openweathermap.org/img/wn/${weatherData.icon}@4x.png`"
          :alt="weatherData.description"
          :title="weatherData.description"
          loading="lazy"
        />
        <span class="temp">{{ Math.round(weatherData.temp) }}°C</span>
      </div>
      <p class="description">{{ capitalizeFirstLetter(weatherData.description) }}</p>
      <div class="weather-details">
        <div class="detail-item">
          <span class="detail-label">Ощущается как:</span>
          <span class="detail-value">{{ Math.round(weatherData.feels_like) }}°C</span>
        </div>
        <div class="detail-item">
          <span class="detail-label">Влажность:</span>
          <span class="detail-value">{{ weatherData.humidity }}%</span>
        </div>
        <div class="detail-item">
          <span class="detail-label">Ветер:</span>
          <span class="detail-value">{{ Math.round(weatherData.wind_speed) }} м/с</span>
        </div>
        <div class="detail-item">
          <span class="detail-label">Давление:</span>
          <span class="detail-value">{{ weatherData.pressure }} гПа</span>
        </div>
      </div>
    </div>

    <canvas ref="confettiCanvas" class="confetti-canvas"></canvas>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue';
import axios from 'axios';
import confetti from 'canvas-confetti';

const API_KEY = import.meta.env.VITE_WEATHER_API_KEY || 'd323032a8c6a1e5005c7097d36436159';
const cityQuery = ref('Минск');
const weatherData = ref(null);
const loading = ref(false);
const error = ref(null);
const confettiCanvas = ref(null);
let confettiInterval = null;

const weatherType = computed(() => {
  if (!weatherData.value) return 'clear';
  
  const desc = weatherData.value.description.toLowerCase();
  const main = weatherData.value.main.toLowerCase();
  
  if (desc.includes('дождь') || main.includes('rain')) return 'rain';
  if (desc.includes('снег') || main.includes('snow')) return 'snow';
  if (desc.includes('облачно') || main.includes('clouds')) return 'clouds';
  if (desc.includes('ясно') || main.includes('clear')) return 'clear';
  if (desc.includes('туман') || main.includes('fog') || main.includes('mist')) return 'fog';
  return 'clear';
});

const weatherTheme = computed(() => {
  return `${weatherType.value}-theme`;
});

const capitalizeFirstLetter = (string) => {
  return string.charAt(0).toUpperCase() + string.slice(1);
};

const startWeatherEffects = () => {
  stopWeatherEffects();
  
  nextTick(() => {
    if (!confettiCanvas.value) return;
    
    const ctx = confettiCanvas.value.getContext('2d');
    confettiCanvas.value.width = window.innerWidth;
    confettiCanvas.value.height = window.innerHeight;
    
    if (weatherType.value === 'rain') {
      confettiInterval = setInterval(() => {
        confetti({
          particleCount: 30,
          angle: 75,
          spread: 100,
          origin: { x: 0, y: 1 },
          colors: ['#a0c4e6', '#7db5e6'],
          shapes: ['circle'],
          gravity: 2,
          ticks: 100,
          scalar: 0.5,
          drift: 1.5
        });
        confetti({
          particleCount: 30,
          angle: 105,
          spread: 100,
          origin: { x: 1, y: 1 },
          colors: ['#a0c4e6', '#7db5e6'],
          shapes: ['circle'],
          gravity: 2,
          ticks: 100,
          scalar: 0.5,
          drift: -1.5
        });
      }, 300);
    } else if (weatherType.value === 'snow') {
      confettiInterval = setInterval(() => {
        confetti({
          particleCount: 20,
          spread: 100,
          origin: { y: 0 },
          colors: ['#ffffff'],
          shapes: ['circle'],
          gravity: 0.5,
          ticks: 200,
          scalar: 0.8,
          drift: 0,
          decay: 0.95
        });
      }, 500);
    }
  });
};

const stopWeatherEffects = () => {
  if (confettiInterval) {
    clearInterval(confettiInterval);
    confettiInterval = null;
  }
  
  if (confettiCanvas.value) {
    const ctx = confettiCanvas.value.getContext('2d');
    ctx.clearRect(0, 0, confettiCanvas.value.width, confettiCanvas.value.height);
  }
};

const fetchWeather = async () => {
  if (!cityQuery.value.trim() || loading.value) return;
  
  loading.value = true;
  error.value = null;
  weatherData.value = null;
  stopWeatherEffects();

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
      icon: weatherResponse.data.weather[0].icon,
      main: weatherResponse.data.weather[0].main.toLowerCase()
    };

    startWeatherEffects();
  } catch (err) {
    error.value = err.response?.data?.message || err.message || 'Произошла ошибка при получении данных';
    console.error('Ошибка при получении погоды:', err);
  } finally {
    loading.value = false;
  }
};

const retryFetch = () => {
  error.value = null;
  fetchWeather();
};

const handleResize = () => {
  if (confettiCanvas.value) {
    confettiCanvas.value.width = window.innerWidth;
    confettiCanvas.value.height = window.innerHeight;
  }
};

onMounted(() => {
  fetchWeather();
  window.addEventListener('resize', handleResize);
});

onUnmounted(() => {
  stopWeatherEffects();
  window.removeEventListener('resize', handleResize);
});
</script>

<style>
/* Стили остаются такими же, как в предыдущем примере, за исключением: */

.confetti-canvas {
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