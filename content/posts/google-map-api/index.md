+++
title = 'Google Map API'
date = 2024-05-27T21:50:10+08:00
draft = false
description = 'Google Map API 使用小筆記'
toc = true
tags = ['JavaScript']
categories = ['JavaScript']
+++
## API Key

Maps JavaScript API 會用到

可以限制Key的

- 使用的API 權限，如Geocoding API
- 平台使用，如限制IP、網站、Android APP、iOS APP

## 簽章私鑰

Map Static API 會用到

如果是一次性需求，可以使用官方的小工具；如果需要動態產生簽名，需要從server side 簽名，官方有提供常見語言的[範例](https://github.com/googlemaps/url-signing)

官方強烈建議加上簽章，如果每日要求超過 25,000 個，就必須提供 API 金鑰和數位簽章

## **Maps JavaScript API**

[document](https://developers.google.com/maps/documentation/javascript/tutorials?hl=zh-tw)

### 載入**Maps JavaScript API**

```jsx
<script>
  (g=>{var h,a,k,p="The Google Maps JavaScript API",c="google",l="importLibrary",q="__ib__",m=document,b=window;b=b[c]||(b[c]={});var d=b.maps||(b.maps={}),r=new Set,e=new URLSearchParams,u=()=>h||(h=new Promise(async(f,n)=>{await (a=m.createElement("script"));e.set("libraries",[...r]+"");for(k in g)e.set(k.replace(/[A-Z]/g,t=>"_"+t[0].toLowerCase()),g[k]);e.set("callback",c+".maps."+q);a.src=`https://maps.${c}apis.com/maps/api/js?`+e;d[q]=f;a.onerror=()=>h=n(Error(p+" could not load."));a.nonce=m.querySelector("script[nonce]")?.nonce||"";m.head.append(a)}));d[l]?console.warn(p+" only loads once. Ignoring:",g):d[l]=(f,...n)=>r.add(f)&&u().then(()=>d[l](f,...n))})({
    key: "YOUR_API_KEY",
    v: "weekly",
    // Use the 'v' parameter to indicate the version to use (weekly, beta, alpha, etc.).
    // Add other bootstrap parameters as needed, using camel case.
  });
</script>
```

### 載入地圖

```jsx
const { Map } = await google.maps.importLibrary("maps");
const map = new Map(document.getElementById("map"), {
  zoom: 18, // 地圖縮放等級
  center: { lat: 43.773145, lng: 11.2559602 }, // 地圖中心
});
```

可以設置地圖類型`mapTypeId`：

- 道路地圖（預設）
- 衛星影像
- 道路+衛星 混合
- 地形

### 標記marker

[example](https://developers.google.com/maps/documentation/javascript/examples/marker-simple)

```jsx
const marker = new google.maps.Marker({
  position: { lat: 43.7731426, lng: 11.2456317 }, // 標記的位置
  map,
  title: "Hello World!", // 游標懸停時顯示的文字
});
```

### 資訊視窗InfoWindow

[example](https://developers.google.com/maps/documentation/javascript/examples/infowindow-simple)

```jsx
const infowindow = new google.maps.InfoWindow({
  content: '<div><h1>聖母百花大教堂</h1>位於義大利<a href="https://zh.wikipedia.org/wiki/佛羅倫斯">佛羅倫斯</a>的一座天主教堂</div>',
});
infowindow.open({anchor: marker, map}); // 配合marker顯示在地圖上
```

### 事件event

地圖可以監聽事件，如點擊、游標移動等

[example](https://developers.google.com/maps/documentation/javascript/examples/event-simple)

範例：監聽點擊事件並取得經緯度

```jsx
map.addListener("click", (mapsMouseEvent) => {
	console.log(mapsMouseEvent.latLng.toJSON());
});
```

上面的latLng.toJSON()印出來長這樣：

```json
{
    "lat": 23.373429728008887,
    "lng": 6.3121497631073
}
```

## Geocoding API

[document](https://developers.google.com/maps/documentation/javascript/geocoding?hl=zh-tw)

### 反向地理編碼

[example](https://developers.google.com/maps/documentation/javascript/examples/geocoding-reverse)

使用 經緯度 取得地理資訊

取得的地理資訊會是一個array，從最相符到最不相符排序

範例：監聽點擊事件並取得經緯度，再用經緯度取得地理資訊

```jsx
map.addListener("click", (mapsMouseEvent) => {
  geocoder
  .geocode({ location: mapsMouseEvent.latLng })
  .then((response) => {
    if (response.results[0]) {
      console.log(response.results[0]);
    } else {
      window.alert("No results found");
    }
  })
  .catch((e) => window.alert("Geocoder failed due to: " + e));
```

上面的response.results[0]印出來長這樣：
```json
    {
        "address_components": [
            {
                "long_name": "臺北車站(忠孝)[雙層巴士]",
                "short_name": "臺北車站(忠孝)[雙層巴士]",
                "types": [
                    "establishment",
                    "point_of_interest",
                    "transit_station"
                ]
            },
            {
                "long_name": "中正區",
                "short_name": "中正區",
                "types": [
                    "administrative_area_level_2",
                    "political"
                ]
            },
            {
                "long_name": "台北市",
                "short_name": "台北市",
                "types": [
                    "administrative_area_level_1",
                    "political"
                ]
            },
            {
                "long_name": "台灣",
                "short_name": "TW",
                "types": [
                    "country",
                    "political"
                ]
            },
            {
                "long_name": "100",
                "short_name": "100",
                "types": [
                    "postal_code"
                ]
            }
        ],
        "formatted_address": "100台灣台北市中正區臺北車站(忠孝)[雙層巴士]",
        "geometry": {
            "location": {
                "lat": 25.04644,
                "lng": 121.517189
            },
            "location_type": "GEOMETRIC_CENTER",
            "viewport": {
                "south": 25.0450910197085,
                "west": 121.5158400197085,
                "north": 25.0477889802915,
                "east": 121.5185379802915
            }
        },
        "place_id": "ChIJkYopX3KpQjQRFWVH5PyqOSM",
        "plus_code": {
            "compound_code": "2GW8+HV 台灣台北市中正區黎明里",
            "global_code": "7QQ32GW8+HV"
        },
        "types": [
            "establishment",
            "point_of_interest",
            "transit_station"
        ]
    }
```

- place_id：地點專屬ID，如商家、地標等都會有place_id，並且可能隨時間改變。參考[官方文件](https://developers.google.com/maps/documentation/places/web-service/place-id?hl=zh-tw)    
- formatted_address：人類可讀地址
- geometry：經緯度、可視區域等

## Map Static API

[document](https://developers.google.com/maps/documentation/maps-static/start?hl=zh-tw)

地圖靜態圖片連結

```
https://maps.googleapis.com/maps/api/staticmap?parameters
```

常用parameters：

- key：API key，必填
- signature：數位簽章
- center：圖片中心，格式為`緯度,精度`
- zoom：縮放等級
- size：圖片尺寸
- format：圖片格式
- scale：影響回傳圖片的像素等級，可用值為1或2
- markers：一或多組mark

## Map URLs

[document](https://developers.google.com/maps/documentation/urls/get-started)

取得某座標和地址的連結

總共有四種格式

- 搜尋：顯示特定地點圖釘的地圖
- 路線
- 顯示地圖：不含標記或路線的地圖
- 街景服務

### 搜尋

```
https://www.google.com/maps/search/?api=1&parameters
```

parameters：

- query：可以為地點名稱、地址或經緯度
- query_place_id：如果只有經緯度沒有place id，則不會在側邊面板顯示地點資訊（如商家、地標等）

## 費用

[document](https://developers.google.com/maps/billing-and-pricing/billing)

每月提供 $200 美元的抵免額
一項產品可能會有多個費率，不同的 SKU（如Maps JavaScript API 的地圖和街景費用不同）
SKU 是按用量分級計費，分為三個級別：

- 0–100,000 次
- 100,001–500,000 次
- 500,001 次以上

費用計算方式：SKU 用量 x 每次使用單價

- [Geocoding API 計費](https://developers.google.com/maps/documentation/geocoding/usage-and-billing?hl=zh-tw)（取得地址等資訊）

| 0–100,000 次 | 100,001–500,000 個 | 500,000 個以上 |
| --- | --- | --- |
| 每個 0.005 美元(每 1,000 個 5.00 美元) | 每個 0.004 美元(每 1,000 個 4.00 美元) | 請https://cloud.google.com/contact-maps?hl=zh-tw洽詢高用量定價資訊 |
- [Maps Static API 計費](https://developers.google.com/maps/documentation/maps-static/usage-and-billing?hl=zh-tw)（靜態圖片）

| 0–100,000 次 | 100,001–500,000 個 | 500,000 個以上 |
| --- | --- | --- |
| 每次 0.002 美元(每 1,000 次 2.00 美元) | 每次 0.0016 美元(每 1,000 次 1.60 美元) | 請https://cloud.google.com/contact-maps?hl=zh-tw洽詢高用量定價資訊 |
- [Maps JavaScript API 計費](https://developers.google.com/maps/documentation/javascript/usage-and-billing?hl=zh-tw)（載入地圖、取得經緯度）

Static Maps 計費

| 0–100,000 次 | 100,001–500,000 次 | 500,000 次以上 |
| --- | --- | --- |
| 每次 $0.007 美元(每 1,000 次 $7.00 美元) | 每次 $0.0056 美元(每 1,000 次 $5.60 美元) | 請https://cloud.google.com/contact-maps?hl=zh-tw洽詢高用量定價資訊 |
- Maps URLs（取得某座標和地址的連結）：不計費

費用計算器： [https://mapsplatform.google.com/pricing/?hl=zh-tw](https://mapsplatform.google.com/pricing/?hl=zh-tw)

## 範例
顯示地圖，點擊地圖上的一點後，console顯示三個連結，分別是：
- 靜態地圖連結
- 地址連結
- 地址連結，含 place_id，會找點擊的地點最近的有place_id的點，因此可能不準確

index.html
```html
<html>
  <head>
    <title>Event Click LatLng</title>
    <script src="https://polyfill.io/v3/polyfill.min.js?features=default"></script>

    <link rel="stylesheet" type="text/css" href="./style.css" />
    <script type="module" src="./index.js"></script>
  </head>
  <body>
    <div id="map"></div>

    <!-- prettier-ignore -->
    <script>(g=>{var h,a,k,p="The Google Maps JavaScript API",c="google",l="importLibrary",q="__ib__",m=document,b=window;b=b[c]||(b[c]={});var d=b.maps||(b.maps={}),r=new Set,e=new URLSearchParams,u=()=>h||(h=new Promise(async(f,n)=>{await (a=m.createElement("script"));e.set("libraries",[...r]+"");for(k in g)e.set(k.replace(/[A-Z]/g,t=>"_"+t[0].toLowerCase()),g[k]);e.set("callback",c+".maps."+q);a.src=`https://maps.${c}apis.com/maps/api/js?`+e;d[q]=f;a.onerror=()=>h=n(Error(p+" could not load."));a.nonce=m.querySelector("script[nonce]")?.nonce||"";m.head.append(a)}));d[l]?console.warn(p+" only loads once. Ignoring:",g):d[l]=(f,...n)=>r.add(f)&&u().then(()=>d[l](f,...n))})
        ({key: "YOUR_API_KEY", v: "weekly"});</script>
  </body>
</html>
```
style.css
```css
#map {
    height: 100%;
}
html,
body {
    height: 100%;
    margin: 0;
    padding: 0;
}
```
index.js
```js
var APIKey = 'YOUR_API_KEY';
var marker;

async function initMap() {
  // Request needed libraries.
  const { Map } = await google.maps.importLibrary("maps");
  const geocoder = new google.maps.Geocoder();
  const map = new Map(document.getElementById("map"), {
    zoom: 18,
    center: { lat: 43.773145, lng: 11.2559602 },
  });
  // Create the initial InfoWindow.
  let infoWindow = new google.maps.InfoWindow();

  // Configure the click listener.
  map.addListener("click", (mapsMouseEvent) => {
    console.log(mapsMouseEvent.latLng.toJSON());
    geocoder
    .geocode({ location: mapsMouseEvent.latLng })
    .then((response) => {
      if (response.results[0]) {
        console.log(response.results[0]);
        // map.setZoom(11);

        if (marker) {
          marker.setPosition(mapsMouseEvent.latLng);
        } else {
          marker = new google.maps.Marker({
            position: mapsMouseEvent.latLng,
            map: map,
          });
        }

        infoWindow.setContent(response.results[0].formatted_address);
        infoWindow.open(map, marker);

        const center = `${mapsMouseEvent.latLng.lat()},${mapsMouseEvent.latLng.lng()}`;
        const picture = `http://maps.googleapis.com/maps/api/staticmap?center=${center}&zoom=${map.getZoom()}&size=800x600&markers=${center}&scale=2&format=jpeg&key=${APIKey}`;

        console.log(picture);

        var url = `https://www.google.com/maps/search/?api=1&query=${center}`;
        console.log(url);
        if (response.results[0].place_id) {
          url += `&query_place_id=${response.results[0].place_id}`;
          console.log(url);
        }
      } else {
        window.alert("No results found");
      }
    })
    .catch((e) => window.alert("Geocoder failed due to: " + e));
  });
}

initMap();
```
