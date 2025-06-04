<script>
  import { onMount } from "svelte";
  import L from "leaflet";
  import 'leaflet/dist/leaflet.css';

  export let level;
  export let changeCondition;
  export let originalColor;
  export let showPopup1;

  let map, geojsonLayer, geojsonLayerDemo;
  let changingLayers = [];
  let mainFeatureLayer = null;
  let countdown = 3;
  let showCountdown = false;
  let animationStarted = false;

  function startCountdown() {
    if (animationStarted) {
        console.warn("⚠ Animation already started, ignoring click.");
        return;
    }
        
    animationStarted = true;
    showCountdown = true;
    document.dispatchEvent(new CustomEvent("startPressed"));

    let interval = setInterval(() => {
        
        if (countdown > 1) {
            countdown -= 1;
        } else {
            clearInterval(interval);
            showCountdown = false;
            startColorChange();
        }
    }, 1000);
  }

  function handleButtonClick() {
    if (animationStarted) {
      alert("Animasjonen kan kun spilles av en gang.");
    } else {
      console.log("The change condition", changeCondition); 
      startCountdown();
    }
  }

  function getColor(color_id) {
    return color_id === 1 ? "#DADADA" : color_id === 2 ? "#A0A0A0" : "#606060";
  }
  function getGeoJSONUrl(level) {
    return `${import.meta.env.BASE_URL}map${level}.geojson`;
  }
  function interpolateColor(color1, color2, progress) {
    const hex = (color) => parseInt(color.slice(1), 16);
    const r1 = (hex(color1) >> 16) & 0xff,
          g1 = (hex(color1) >> 8) & 0xff,
          b1 = hex(color1) & 0xff;
    const r2 = (hex(color2) >> 16) & 0xff,
          g2 = (hex(color2) >> 8) & 0xff,
          b2 = hex(color2) & 0xff;

    return `#${((1 << 24) + (Math.round(r1 + (r2 - r1) * progress) << 16) + 
                  (Math.round(g1 + (g2 - g1) * progress) << 8) + 
                  Math.round(b1 + (b2 - b1) * progress)).toString(16).slice(1)}`;
  }
  function animateFeatureColor(layer, startColor, endColor, duration, callback) {
    const startTime = performance.now();
    function step(currentTime) {
      let progress = (currentTime - startTime) / duration;
      if (progress > 1) progress = 1;
      const newColor = interpolateColor(startColor, endColor, progress);
      layer.setStyle({ fillColor: newColor });

      if (progress < 1) {
        requestAnimationFrame(step);
      } else {
        if (callback) callback();
      }
    }
    requestAnimationFrame(step);
  }
  function getRandomGrayColor(excludeColor) {
    const grayColors = ["#DADADA", "#A0A0A0", "#606060"];
    return grayColors.filter(color => color !== excludeColor)[Math.floor(Math.random() * 2)];
  }

  function startColorChange() {
  if (changingLayers.length === 0) {
    console.warn("⚠ No changing features found!");
    return;
  }

  changingLayers.forEach(({ layer, startColor, endColor }) => {
    animateFeatureColor(layer, startColor, endColor, 2000);
  });

  setTimeout(() => {
    if (mainFeatureLayer) {
      addRedOutline(mainFeatureLayer);
      // Dispatch an event to indicate the red square has been shown
      document.dispatchEvent(new CustomEvent("redSquareShown"));
    }
  }, 2200);
  }
  async function loadGeoJSON(level) {
    try {
      let response = await fetch(getGeoJSONUrl(level));
      let data = await response.json();
      if (geojsonLayer) geojsonLayer.clearLayers();
      changingLayers = [];

      geojsonLayer = L.geoJSON(data, {
        style: feature => ({
          fillColor: getColor(feature.properties.color_id),
          weight: 1,
          color: "white",
          fillOpacity:1.0
          //fillOpacity: feature.properties.changing ? 1.0 : 0.9
        }),
        onEachFeature: function (feature, layer) {
          const isMain = feature.properties.mainFeature;
          const isChanging = feature.properties.changing;
          const startColor = getColor(feature.properties.color_id);
          const endColor = getColor(feature.properties.new_color_id);

          // Store reference if it's the main feature
          if (isMain) {
          mainFeatureLayer = layer;
          originalColor = startColor;

          if (changeCondition === "Change") {
            changingLayers.push({
              layer,
              startColor,
              endColor
            });
          }
        } else if (isChanging) {
          changingLayers.push({
            layer,
            startColor,
            endColor
          });
        }
      }
      }).addTo(map);


      map.fitBounds(geojsonLayer.getBounds());
    } catch (error) {
      console.error("❌ Error loading GeoJSON:", error);
    }
  }

  function addRedOutline(layer) {
    const bounds = layer.getBounds();
    L.rectangle(bounds, { color: "#FF0000", weight: 2, fillOpacity: 0 }).addTo(map);
  }


  onMount(() => {
    map = L.map("map", { zoomControl: false, attributionControl: false }).setView([0, 0], 2);
    loadGeoJSON(level);
    
    document.addEventListener("resetAnimation", () => {
        animationStarted = false;
        showCountdown = false;
        countdown = 3;
      
    });
  
      const legendControl = L.control({ position: 'topleft' });
      legendControl.onAdd = function(map) {
        // Create a container for the legend (Leaflet will append this container inside the map)
        const div = L.DomUtil.create('div', 'info legend');
        div.innerHTML = `
          <h4 style="margin-bottom:5px;">Klasser</h4>
          <i style="background: ${getColor(1)}; width:18px; height:18px; display:inline-block; margin-right:5px;"></i> Lav<br>
          <i style="background: ${getColor(2)}; width:18px; height:18px; display:inline-block; margin-right:5px;"></i> Medium<br>
          <i style="background: ${getColor(3)}; width:18px; height:18px; display:inline-block; margin-right:5px;"></i> Høy<br>
        `;
        return div;
      };
      legendControl.addTo(map);
});
  
$: if (level) {
    changeCondition = Math.random() < 0.5 ? "Change" : "No change";
    console.log("🎲 New random change condition:", changeCondition);
    document.dispatchEvent(new CustomEvent('next', { detail: { changeCondition } }));
    loadGeoJSON(level);
  }
  $: console.log("Map level prop is:", level);
  $:  loadGeoJSON(level);


</script>

<div class="map-container">
  <div id="map">
    {#if showCountdown}
      <p class="countdown-overlay">{countdown}...</p>
    {/if}

</div>

<!--   <div class="description-container">
    <p>
      Observer kartet, noen regioner vil <strong>skifte farge</strong> etter du har trykket start.  
     Etter fargeskiftet vil en <strong>rød ramme</strong> fremheve én region.   
    </p>
    <p class="warning-text">
      ⚠ <strong>Ikke refresh siden</strong>, da starter studien på nytt.
    </p>
  </div> -->
  {#if !showPopup1}
  <button 
    class="start-animation-btn {animationStarted ? 'disabled' : ''}" 
    on:click={handleButtonClick}
    title={animationStarted ? "Animasjonen kan kun spilles av en gang." : ""}
  >
    {animationStarted ? "Animasjon ferdig" : "Start animasjon"}
  </button>
{/if}

</div>

<style>
  .map-container {
    position: relative;
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-top: 5px;
  }
  h3{
    margin-block-start: 0em;
    margin-block-end: 0em
  }

  #map {
    width: 600px;
    height: 400px;
    background: white;
    border-radius: 8px;
    position: relative;
  }
 .description-container {
    background: #eef1f6;
    padding: 10px;
    padding-left: 50px;
    padding-right: 50px;
    max-width: 580px;

    border-radius: 6px;
    margin-top: 10px;
    text-align: center;
    font-size: 14px;
    margin-bottom: 5px;
    font-size: 14px;
    align-items: center;
    text-emphasis: none;
    max-width: 460px;
  } 


  .countdown-overlay {
    position: absolute;
    top: 90%;
    left: 20%;
    transform: translate(-50%, -50%);
    font-size: 14px;
    font-weight: bold;
    color: rgb(0, 0, 0);
    background: rgba(255, 255, 255, 0.8);
    padding: 10px;
    border-radius: 6px;
  }

  .start-animation-btn {
    margin-bottom: 10px;
    margin-top: 6px;
    padding: 10px 15px;
    z-index: 10;  
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-size: 13px;
  }

  .start-animation-btn:disabled {
    background-color: gray;
    cursor: not-allowed;
  }
  .start-animation-btn {
    margin-bottom: 10px;
    margin-top: 6px;
    padding: 10px 15px;
    z-index: 10;  
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
    font-size: 13px;
    transition: background-color 0.3s;
  }

  .start-animation-btn.disabled {
    background-color: gray;
    cursor: not-allowed;
  }
</style>
