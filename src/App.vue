<template>
    <div class="container">
        <h1> Public Holidays </h1>
        <div class="selects">
            <select v-model="selectedCountry" @change="fetchProvinces">
                <option value="">---Please select a country---</option>
                <option v-for="country in countries" :key="country.countryCode" :value="country.countryCode">
                    {{ country.countryName }}
                </option>
            </select>

            <select v-model="selectedProvince" @change="fetchCities">
                <option value="">---Please select a state/province---</option>
                <option v-for="province in provinces" :key="province.geonameId" :value="province.geonameId">
                    {{ province.name }}
                </option>
            </select>

            <select v-model="selectedCity">
                <option value="">---Please select a city---</option>
                <option v-for="city in cities" :key="city.geonameId" :value="city.geonameId">
                    {{ city.name }}
                </option>
            </select>
        </div>

        <div class="holiday-weather">
            <div v-if="holidays.length">
                <h2>Holidays:</h2>
                <ul>
                    <li v-for="holiday in holidays" :key="holiday.date">
                        <input type="radio" :id="holiday.date" :value="holiday" v-model="selectedHoliday" @change="fetchWeather(selectedHoliday)">
                        <label :for="holiday.date">{{ holiday.date }}: {{ holiday.name }}</label>
                    </li>
                </ul>
            </div>

            <div v-if="weather">
                <h2>Weather:</h2>
                <p>{{ weather }}</p>
            </div>
        </div>

        <div class="stay22">
            <!-- Stay22 iframe -->
            <iframe :src="stay22Url" id="stay22-widget" width="100%" height="1000" frameborder="0"></iframe>
        </div>

    </div>
</template>

<style scoped>
.container {
    display: flex;
    flex-direction: column;
    gap: 2rem;
    width: 100%;
}

.container > * {
    width: 100%;
}
</style>

<script>


import { ref, watchEffect } from 'vue'
import { differenceInDays } from 'date-fns'
import axios from 'axios'

export default {
    setup() {
        const username = 'lya_mira';

        const selectedCountry = ref('')
        const selectedProvince = ref('')
        const selectedCity = ref('')

        const countries = ref([])
        const provinces = ref([])
        const cities = ref([])

        const fetchCountries = async () => {
            const response = await fetch(`http://api.geonames.org/countryInfoJSON?username=${username}`)
            const data = await response.json()
            countries.value = data.geonames
        }

        const fetchProvinces = async () => {
            const country = countries.value.find(c => c.countryCode === selectedCountry.value)
            if (country) {
                const response = await fetch(`http://api.geonames.org/childrenJSON?geonameId=${country.geonameId}&username=${username}`)
                const data = await response.json()
                provinces.value = data.geonames
            } else {
                provinces.value = []
            }
            selectedProvince.value = ''
            selectedCity.value = ''
        }

        const fetchCities = async () => {
            const province = provinces.value.find(p => p.geonameId === selectedProvince.value)
            if (province) {
                const response = await fetch(`http://api.geonames.org/childrenJSON?geonameId=${province.geonameId}&username=${username}`)
                const data = await response.json()
                cities.value = data.geonames
            } else {
                cities.value = []
            }
            selectedCity.value = ''
        }

        watchEffect(() => {
            fetchCountries()
        })

        const holidays = ref([])

        const fetchHolidays = async () => {
            if (selectedCountry.value) {
                const country = countries.value.find(c => c.countryCode === selectedCountry.value)
                if (country) {
                    const year = new Date().getFullYear()
                    const response = await fetch(`https://date.nager.at/api/v3/publicholidays/${year}/${country.countryCode}`)
                    const data = await response.json()
                    holidays.value = data
                }
            }
        }

        watchEffect(() => {
            fetchHolidays()
        })

        const selectedHoliday = ref(null)
        const weather = ref(null)

        const fetchWeather = async (holiday) => {
            try {
                if (selectedCity.value && holiday) {
                    const city = cities.value.find(c => c.geonameId === selectedCity.value)
                    if (!city) {
                        weather.value = 'Latitude and longitude information for the selected city was not found'
                        return
                    }
                    const date = new Date(holiday.date)
                    const today = new Date()
                    const daysDifference = differenceInDays(today, date)
                    let response


                    if (daysDifference > 0) {
                        response = await fetch(`https://api.openweathermap.org/data/3.0/onecall/timemachine?lat=${city.lat}&lon=${city.lng}&dt=${Math.floor(date.getTime() / 1000)}&units=metric&appid=2805fc2f1e5290a6243c6049b4be1626`)
                        const data = await response.json()
                        weather.value =
                            "Description: " + data.data[0].weather[0].description +
                            "  Temperature: " + data.data[0].temp + "°C" +
                            "  Humidity: " + data.data[0].humidity + "%" +
                            "  Wind Speed: " + data.data[0].wind_speed + "m/s"
                    } else if (daysDifference <= 0 && daysDifference >= -7) {
                        response = await fetch(`https://api.openweathermap.org/data/3.0/onecall?lat=${city.lat}&lon=${city.lng}&exclude=current,minutely,hourly,alerts&units=metric&appid=2805fc2f1e5290a6243c6049b4be1626`)
                        const data = await response.json()
                        weather.value =
                            "Description: " + data.daily[-daysDifference].weather[0].description +
                            "  Temperature: " + data.daily[-daysDifference].temp.day + "°C" +
                            "  Humidity: " + data.daily[-daysDifference].humidity + "%" +
                            "  Wind Speed: " + data.daily[-daysDifference].wind_speed + "m/s"
                    } else {
                        weather.value = 'Weather information for the selected date is not available'
                        return
                    }
                }
            } catch (error) {
                console.log(error)
                weather.value = 'Error getting weather information'
            }
        }

        watchEffect(() => {
            if (selectedHoliday.value) {
                fetchWeather(selectedHoliday.value)
            }
        })

        const stay22Url = ref('')

        watchEffect(() => {
            if (selectedHoliday.value) {
                fetchWeather(selectedHoliday.value)
                updateStay22Url()
            }
        })

        const updateStay22Url = () => {
            if (selectedHoliday.value && selectedCity.value) {
                const city = cities.value.find(c => c.geonameId === selectedCity.value)
                if (!city) {
                    stay22Url.value = ''
                    return
                }

                const date = new Date(selectedHoliday.value.date)
                const timestamp = Math.floor(date.getTime() / 1000)
                stay22Url.value = `https://www.stay22.com/embed/gm?aid=affiliateid&lat=${city.lat}&lng=${city.lng}&eventstart=${timestamp}&isnear=false`
            }
        }

        return {
            selectedCountry,
            selectedProvince,
            selectedCity,
            countries,
            provinces,
            cities,
            fetchProvinces,
            fetchCities,
            fetchHolidays,
            holidays,
            selectedHoliday,
            fetchWeather,
            weather,
            stay22Url
        }
    }
}
</script>
