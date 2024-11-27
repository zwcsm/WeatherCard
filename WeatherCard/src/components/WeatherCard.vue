<template>
  <div class="weather-card" :class="{ 'loading': isLoading, [weatherClass]: !isLoading }">
    <template v-if="isLoading">
      <div class="loading-text">
        <span class="loading-icon"></span>
        加载中...
      </div>
    </template>
    <template v-else>
      <div class="temperature" v-if="temperature">
        {{ temperature }}°
      </div>
      <div class="info" v-if="location">
        <div class="location">
          {{ location }}
        </div>
        <div class="description" v-if="description">
          <span class="des">{{ description }}</span>
          <span class="temp-range" v-if="highTemp && lowTemp">{{ highTemp }}° ~ {{ lowTemp }}°</span>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';

const apiKey = '8b186e3e7402a730193164e07a0a4a9e';
const location = ref('');
const description = ref('');
const temperature = ref('');
const highTemp = ref('');
const lowTemp = ref('');
const isLoading = ref(true);

const weatherClass = computed(() => {
  const classList = {
    '晴': 'sunny',
    '多云': 'cloudy',
    '阴': 'overcast',
    '小雨': 'rainy',
    '中雨': 'rainy',
    '大雨': 'rainy',
    '暴雨': 'rainy',
    '雷阵雨': 'stormy',
    '雪': 'snowy',
    '雷': 'stormy',
    '风': 'windy'
  };
  return classList[description.value] || 'default';
});

const getCachedData = (key) => {
  try {
    const cachedData = localStorage.getItem(key);
    if (cachedData) {
      const { data, timestamp } = JSON.parse(cachedData);
      if (Date.now() - timestamp < 1000 * 60 * 10) { // 10 minutes
        return data;
      }
    }
  } catch (error) {
    console.error('Error reading cache:', error);
  }
  return null;
};

const fetchWeatherData = async (cityCode) => {
  isLoading.value = true;

  try {
    const baseUrl = 'https://restapi.amap.com/v3/weather/weatherInfo';
    const baseParams = new URLSearchParams({
      city: cityCode,
      key: apiKey,
      output: 'JSON'
    });

    const [currentResponse, forecastResponse] = await Promise.all([
      fetch(`${baseUrl}?${baseParams}&extensions=base`),
      fetch(`${baseUrl}?${baseParams}&extensions=all`)
    ]);

    const currentData = await currentResponse.json();
    const forecastData = await forecastResponse.json();

    if (currentData.status === '1' && currentData.lives?.[0]) {
      const weatherData = currentData.lives[0];
      location.value = weatherData.city;
      description.value = weatherData.weather;
      temperature.value = weatherData.temperature;
    }

    if (forecastData.status === '1' && forecastData.forecasts?.[0]?.casts?.[0]) {
      const todayForecast = forecastData.forecasts[0].casts[0];
      highTemp.value = todayForecast.daytemp;
      lowTemp.value = todayForecast.nighttemp;
    }
  } catch (error) {
    console.error('Error fetching weather data:', error);
  } finally {
    setTimeout(() => {
      isLoading.value = false;
    }, 500);
  }
};

onMounted(async () => {
  isLoading.value = true;
  try {
    await fetchWeatherData('330100');
  } catch (error) {
    console.error('Error:', error);
  }
});
</script>

<style scoped>
.weather-card {
  display: block;
  height: 48px;
  width: 172px;
  padding: 12px 16px;
  border-radius: 20px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
  color: rgb(0, 0, 0);
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  cursor: pointer;
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.weather-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 6px 12px rgba(0, 0, 0, 0.15);
}

.weather-card.loading {
  display: flex;
  justify-content: center;
  align-items: center;
}

.loading-text {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 16px;
  color: #333;
}

.loading-icon {
  display: inline-block;
  width: 20px;
  height: 20px;
  border: 3px solid rgba(0, 0, 0, 0.3);
  border-top-color: #333;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.temperature {
  float: left;
  font-size: 28px;
  line-height: 48px;
  margin-right: 16px;
  font-weight: bold;
}

.info {
  position: relative;
  float: left;
  width: 100px;
  height: 48px;
}

.location {
  height: 24px;
  float: left;
  font: 14px/26px "Helvetica Neue", Arial, sans-serif;
}

.description {
  float: left;
  position: absolute;
  top: 24px;
  left: 0;
  display: block;
  height: 24px;
  font: 12px/22px "Helvetica Neue", Arial, sans-serif;
  color: #000000;
}

.des {
  float: left;
  margin-right: 4px;
}

.temp-range {
  float: left;
  margin-left: auto;
  font-size: 14px;
  opacity: 0.9;
}

/* Weather-specific styles */
.sunny {
  background: linear-gradient(45deg, #d5e5ff 40%, #fff0d1);
}

.cloudy {
  background: linear-gradient(45deg, #b8c7d4, #dee3ef);
}

.overcast {
  background: linear-gradient(45deg, #708090, #2F4F4F);
  color: #fff;
}

.rainy {
  background: linear-gradient(45deg, #85929d, #c2c9d1);
  color: #fff;
}

.snowy {
  background: linear-gradient(45deg, #B0E0E6,#e1f1ff);
}

.stormy {
  background: linear-gradient(45deg, #85929d, #c2c9d1);
  color: #fff;
}

.windy {
  background: linear-gradient(45deg, #b8c7d4, #dee3ef);
  color: #fff;
}

.default {
  background: linear-gradient(45deg, #b8c7d4, #dee3ef);
}
</style>