<!-- 要求实现展示天气对应的Icon图标，可以做相对粗略的实现（比如暴雨、
 大雨、中雨、小雨、阵雨都对应展示“雨”图标）； 

要求实现天气卡片的响应式设计，即天气卡片在移动端和PC端均获得美观的
展示，不可缺失信息； 

完成以上需求后，将Vue项目部署在Vercel中（需要翻墙，看个人意愿），
或者你有想法有能力的话，也可以购买并部署在个人云服务器上，
最终目的是让你的项目可以在公网被访问到； -->

<template>
  <div class="weather-card" :class="{ 'loading': isLoading, [weatherClass]: !isLoading }" @click="openLocationModal">
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

  <!-- Location-->
  <div v-if="showLocationModal" class="modal">
    <div class="modal-content">
      <button @click="closeLocationModal">×</button>
      <p>切换地区</p>
      <input v-model="searchQuery" @input="searchLocations" placeholder="搜索" />
      <ul>
        <li v-for="city in filteredCities" :key="city.adcode" @click="selectLocation(city)">
          {{ city.name }}
        </li>
      </ul>
    </div>
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
const showLocationModal = ref(false);
const searchQuery = ref('');
const cities = ref([]);

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

const filteredCities = computed(() => {
  if (!searchQuery.value) return [];
  return cities.value.filter(city => 
    city.name.toLowerCase().includes(searchQuery.value.toLowerCase())
  );
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

const setCachedData = (key, data) => {
  localStorage.setItem(key, JSON.stringify({ data, timestamp: Date.now() }));
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

    const weatherData = {
      location: location.value,
      description: description.value,
      temperature: temperature.value,
      highTemp: highTemp.value,
      lowTemp: lowTemp.value
    };
    setCachedData(`weather_${cityCode}`, weatherData);
  } catch (error) {
    console.error('Error fetching weather data:', error);
  } finally {
    setTimeout(() => {
      isLoading.value = false;
    }, 500);
  }
};

const openLocationModal = () => {
  showLocationModal.value = true;
};

const closeLocationModal = () => {
  showLocationModal.value = false;
  searchQuery.value = '';
};

const searchLocations = async () => {
  if (searchQuery.value.length < 1) return;
  
  const url = `https://restapi.amap.com/v3/config/district?keywords=${searchQuery.value}&subdistrict=0&key=${apiKey}`;
  try {
    const response = await fetch(url);
    const data = await response.json();
    if (data.status === '1' && data.districts) {
      cities.value = data.districts.map(district => ({
        name: district.name,
        adcode: district.adcode
      }));
    }
  } catch (error) {
    console.error('Error searching locations:', error);
  }
};

const selectLocation = (city) => {
  fetchWeatherData(city.adcode);
  closeLocationModal();
};

onMounted(async () => {
  isLoading.value = true;
  const cachedWeatherData = getCachedData('weather_330100');
  
  if (cachedWeatherData) {
    location.value = cachedWeatherData.location;
    description.value = cachedWeatherData.description;
    temperature.value = cachedWeatherData.temperature;
    highTemp.value = cachedWeatherData.highTemp;
    lowTemp.value = cachedWeatherData.lowTemp;
    isLoading.value = false;
  } else {
    try {
      await fetchWeatherData('330100');
    } catch (error) {
      console.error('Error:', error);
    }
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
  background: linear-gradient(45deg, #b8c7d4, #dee3ef);
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

.modal {
  position: fixed;
  z-index: 1;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  overflow: auto;
  background-color: rgba(0,0,0,0.4);
  display: flex;
  justify-content: center;
  align-items: center;
}

.modal-content {
  background: linear-gradient(0deg,#ffe9e9, #c3daff);
  padding: 20px;
  border-radius: 10px;
  height: 700px;
  width: 450px;
}

.modal-content p {
  margin: 0;
  margin-left: 190px;
  color: #8d8d8d;
}

.modal-content input {
  margin-top: 35px;
  width: 430px;
  height: 20px;
  border: 1px solid #dddddd;
  border-radius:10px;
  padding: 10px;
}

.modal-content input:focus {
  outline: none;
  border: 2px solid #5f5f5f;
}

.modal-content ul {
  margin-top: 20px;
  list-style-type: none;
  padding: 0;
  max-height: 550px;
  overflow-y: auto;
}

.modal-content ul::-webkit-scrollbar {
     width: 8px;
 }
.modal-content ul::-webkit-scrollbar-track {
     background-color: #f1f1f1;
 }
.modal-content ul::-webkit-scrollbar-thumb {
     background-color: #888;
     border-radius: 4px;
 }
.modal-content ul::-webkit-scrollbar-thumb:hover {
     background-color: #969696;
 }

.modal-content li {
  margin: 5px 0;
  padding: 10px;
  border-radius:10px;
  cursor: pointer;
}

.modal-content li:hover {
  background-color: #ffffff;
}

.modal-content button {
  float: left;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background-color: rgb(255 115 48);
  color: white;
  border: none;
  cursor: pointer;
}

.modal-content button:hover {
  background-color: #f66d2e;
}

/* Weather-specific styles */
.sunny {
  background: linear-gradient(45deg, #d5e5ff 45%, #fff0d1);
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