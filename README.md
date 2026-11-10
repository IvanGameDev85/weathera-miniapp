<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Weathera — Погода</title>
<script src="https://telegram.org/js/telegram-web-app.js"></script>
<style>
*{box-sizing:border-box}
body{margin:0;background:#091426;color:#fff;font-family:Arial,sans-serif;padding:20px}
.app{max-width:520px;margin:auto}
.logo{font-size:27px;font-weight:bold;margin:12px 0 22px}
.logo span{color:#69c8ff}
.search{display:flex;gap:8px;margin-bottom:20px}
input{min-width:0;flex:1;padding:14px;border:0;border-radius:13px;background:#192a42;color:white;font-size:16px}
button{border:0;border-radius:12px;padding:12px;background:#328bd2;color:white;font-weight:bold}
.card{background:linear-gradient(145deg,#2379ba,#294c9c);border-radius:24px;padding:24px;margin-bottom:16px}
.city{font-size:19px}
.temp{font-size:68px;font-weight:bold;margin:15px 0}
.desc{font-size:19px}
.details{display:grid;grid-template-columns:repeat(3,1fr);gap:9px;margin-top:24px}
.detail{background:#ffffff20;border-radius:13px;padding:12px 7px;text-align:center;font-size:13px}
.detail b{display:block;font-size:17px;margin-top:7px}
h3{margin:25px 0 13px}
.hourly{display:flex;overflow:auto;gap:10px}
.hour{background:#192a42;border-radius:14px;padding:14px;text-align:center;min-width:76px}
.day{display:flex;justify-content:space-between;padding:15px;background:#192a42;border-radius:13px;margin-bottom:8px}
.muted{color:#b7c8dc;font-size:13px}
.action{width:100%;margin:5px 0 15px;padding:14px}
#status{color:#b7c8dc;font-size:13px;margin:10px 0}
</style>
</head>
<body>
<div class="app">
 <div class="logo">☀️ Weather<span>a</span></div>
 <div class="search">
  <input id="cityInput" placeholder="Введите город" value="Волгоград">
  <button onclick="searchCity()">Найти</button>
 </div>
 <button class="action" onclick="useLocation()">📍 Моё местоположение</button>
 <div id="status">Загружаем погоду...</div>
 <div class="card">
  <div class="city" id="city">Волгоград</div>
  <div class="temp" id="temp">--°</div>
  <div class="desc" id="desc">Загрузка...</div>
  <div class="muted" id="feels">Ощущается как --°</div>
  <div class="details">
   <div class="detail">Влажность<b id="humidity">--%</b></div>
   <div class="detail">Ветер<b id="wind">--</b></div>
   <div class="detail">Осадки<b id="rain">--</b></div>
  </div>
 </div>
 <h3>Почасовой прогноз</h3>
 <div class="hourly" id="hourly"></div>
 <h3>На 7 дней</h3>
 <div id="days"></div>
 <p class="muted">Данные о погоде: Open-Meteo</p>
</div>
<script>
const tg=window.Telegram?.WebApp;
if(tg){tg.ready();tg.expand();}
let currentCity="Волгоград";
function weatherText(c){
 if(c===0)return "Ясное небо ☀️";
 if([1,2].includes(c))return "Переменная облачность 🌤️";
 if(c===3)return "Облачно ☁️";
 if([45,48].includes(c))return "Туман 🌫️";
 if([51,53,55,61,63,65,80,81,82].includes(c))return "Дождь 🌧️";
 if([71,73,75,77,85,86].includes(c))return "Снег ❄️";
 if([95,96,99].includes(c))return "Гроза ⛈️";
 return "Переменная погода";
}
async function loadWeather(lat,lon,name){
 document.getElementById("status").textContent="Обновляем прогноз...";
 try{
 const url=`https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,relative_humidity_2m,apparent_temperature,is_day,precipitation,weather_code,wind_speed_10m&hourly=temperature_2m,precipitation_probability,weather_code&daily=weather_code,temperature_2m_max,temperature_2m_min,precipitation_probability_max&forecast_days=7&timezone=auto`;
 const r=await fetch(url);if(!r.ok)throw Error("Ошибка загрузки");
 const d=await r.json(),c=d.current;
 document.getElementById("city").textContent=name;
 document.getElementById("temp").textContent=Math.round(c.temperature_2m)+"°";
 document.getElementById("desc").textContent=weatherText(c.weather_code);
 document.getElementById("feels").textContent="Ощущается как "+Math.round(c.apparent_temperature)+"°";
 document.getElementById("humidity").textContent=c.relative_humidity_2m+"%";
 document.getElementById("wind").textContent=Math.round(c.wind_speed_10m)+" км/ч";
 document.getElementById("rain").textContent=c.precipitation+" мм";
 let h="";
 let start=d.hourly.time.findIndex(t=>t>=c.time);
 for(let i=Math.max(0,start);i<Math.min(d.hourly.time.length,start+12);i++){
 h+=`<div class="hour"><div class="muted">${d.hourly.time[i].slice(11,16)}</div><p>${weatherText(d.hourly.weather_code[i]).split(" ")[0]}</p><b>${Math.round(d.hourly.temperature_2m[i])}°</b><div class="muted">💧 ${d.hourly.precipitation_probability[i]??0}%</div></div>`;
 }
 document.getElementById("hourly").innerHTML=h;
 let days="";
 const names=["Вс","Пн","Вт","Ср","Чт","Пт","Сб"];
 d.daily.time.forEach((date,i)=>{
 const dt=new Date(date+"T12:00:00");
 days+=`<div class="day"><span>${i===0?"Сегодня":names[dt.getDay()]+" "+date.slice(8,10)+"."+date.slice(5,7)}<br><small class="muted">${weatherText(d.daily.weather_code[i])}</small></span><b>${Math.round(d.daily.temperature_2m_max[i])}° / ${Math.round(d.daily.temperature_2m_min[i])}°</b></div>`;
 });
 document.getElementById("days").innerHTML=days;
 document.getElementById("status").textContent="Прогноз обновлён";
 }catch(e){document.getElementById("status").textContent="Не удалось загрузить погоду. Проверьте интернет и попробуйте снова.";}
}
async function searchCity(){
 const name=document.getElementById("cityInput").value.trim();
 if(!name)return;
 document.getElementById("status").textContent="Ищем город...";
 try{
 const r=await fetch("https://geocoding-api.open-meteo.com/v1/search?name="+encodeURIComponent(name)+"&count=1&language=ru&format=json");
 const d=await r.json();
 if(!d.results?.length)throw Error("Город не найден");
 const p=d.results[0];
 currentCity=p.name;
 loadWeather(p.latitude,p.longitude,p.name+(p.country?", "+p.country:""));
 }catch(e){document.getElementById("status").textContent="Город не найден. Попробуйте другое название.";}
}
function useLocation(){
 if(!navigator.geolocation){document.getElementById("status").textContent="Геолокация недоступна";return;}
 navigator.geolocation.getCurrentPosition(async p=>{
 const lat=p.coords.latitude,lon=p.coords.longitude;
 loadWeather(lat,lon,"Моё местоположение");
 },()=>document.getElementById("status").textContent="Разрешите доступ к местоположению.");
}
searchCity();
</script>
</body>
</html>