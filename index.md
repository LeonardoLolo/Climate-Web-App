```js

// Importando librerias
import * as L from "npm:leaflet";
import { FileAttachment } from "npm:@observablehq/stdlib";
import * as Plot from "npm:@observablehq/plot";
import * as d3 from "npm:d3";

// Mapeo de variables a índice en el CSV
const variableIndexMap = {
  tavg: 1, tmin: 2, tmax: 3, prcp: 4, snow: 5,
  wdir: 6, wspd: 7, wpgt: 8, pres: 9, tsun: 10
};
const variableLabels = {
  tavg: "Temperatura Promedio (°F)",
  tmin: "Temperatura Mínima (°F)",
  tmax: "Temperatura Máxima (°F)",
  prcp: "Precipitación Diaria (mm)",
  snow: "Profundidad de Nieve (mm)",
  wdir: "Dirección del Viento (°)",
  wspd: "Velocidad del Viento (m/s)",
  wpgt: "Ráfaga de Viento Máx. (m/s)",
  pres: "Presión Atmosférica (hPa)",
  tsun: "Insolación (min)"
};

let variableSeleccionada = "tavg";
let currentFilteredData = [];
let lastStationID = "";
let lastStationName = "";

const container = display(document.createElement("div"));
container.style = `
  display: flex; 
  flex-direction: column; 
  width: 125%; 
  overflow: visible;`;

// Creando el header
const header = document.createElement("div");
header.style = `
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 15px 40px;
  font-family: sans-serif;
  flex-wrap: wrap;
  gap: 10px;
  background-color: var(--header-bg);
  color: var(--header-color);
  border-bottom: 2px solid var(--header-border);
`;

// Creando el logo izquierdo
const logoLeft = document.createElement("img");
logoLeft.src = "https://cdat.uprh.edu/~leonardo.morales2/UPRH_logo";
logoLeft.style = "height: 70px; max-width: 180px; object-fit: contain;";

// Definiendo el contenedor de los titulos
const titleContainer = document.createElement("div");
titleContainer.style = `
  flex-grow: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-width: 300px;
  text-align: center;
`;

// Titulo principal del web app
const title = document.createElement("h1");
title.textContent = "Aplicación Web para la Exploración y Descarga de Datos Meteorológicos Globales";
title.style = "margin: 0; font-size: 18px; color: var(--header-color);";

// Nombre del autor y fuente de datos
const author = document.createElement("div");
author.innerHTML = `por Leonardo Morales Casillas<br><span style="font-size: 12px;">Fuente: <a href="https://meteostat.net/es/" target="_blank" style="color: var(--header-subtext); text-decoration: underline;">Meteostat</a></span>`;
author.style = "font-size: 13px; color: var(--header-subtext); text-align: center;";

titleContainer.appendChild(title);
titleContainer.appendChild(author);

// Definiendo el logo derecho
const logoRight = document.createElement("img");
logoRight.src = "https://cdat.uprh.edu/~leonardo.morales2/CDAT_logo";
logoRight.style = "height: 70px; max-width: 180px; object-fit: contain;";

const githubLink = document.createElement("a");
githubLink.href = "https://github.com/LeonardoLolo/Climate-Web-App/blob/main/index.md";
githubLink.target = "_blank";
githubLink.rel = "noopener noreferrer";
githubLink.title = "Visita mi GitHub";

// Crear imagen del logo
const githubIcon = document.createElement("img");

// Determinar si el tema es oscuro
const prefersDark = window.matchMedia("(prefers-color-scheme: dark)").matches;

githubIcon.src = prefersDark
  ? "https://cdat.uprh.edu/~leonardo.morales2/githubWhite_logo" // Logo blanco
  : "https://raw.githubusercontent.com/devicons/devicon/master/icons/github/github-original.svg"; // Logo oscuro

githubIcon.alt = "GitHub";
githubIcon.style = `
  height: 42px;
  width: 42px;
  object-fit: contain;
  margin-left: 10px;
  transition: transform 0.2s;
  cursor: pointer;
`;

// Actualiza el icono cuando se cambia de color el background
window.matchMedia("(prefers-color-scheme: dark)").addEventListener("change", e => {
  githubIcon.src = e.matches
    ? "https://cdat.uprh.edu/~leonardo.morales2/githubWhite_logo"
    : "https://raw.githubusercontent.com/devicons/devicon/master/icons/github/github-original.svg";
});

githubIcon.addEventListener("mouseenter", () => githubIcon.style.transform = "scale(1.1)");
githubIcon.addEventListener("mouseleave", () => githubIcon.style.transform = "scale(1)");

githubLink.appendChild(githubIcon);
header.appendChild(logoLeft);
header.appendChild(githubLink);
header.appendChild(titleContainer);
header.appendChild(logoRight);
container.appendChild(header);

// Tema adaptativo oscuro/claro
const root = document.documentElement;

if (prefersDark) {
  root.style.setProperty('--header-bg', '#1a1a1a');
  root.style.setProperty('--header-color', 'white');
  root.style.setProperty('--header-border', '#444');
  root.style.setProperty('--header-subtext', '#ccc');
} else {
  root.style.setProperty('--header-bg', '#f0f0f0');
  root.style.setProperty('--header-color', '#111');
  root.style.setProperty('--header-border', '#ccc');
  root.style.setProperty('--header-subtext', '#444');
}

//  Contenedor de las opciones del grafico
const layoutContainer = document.createElement("div");
layoutContainer.style = `
  display: flex; 
  height: 650px; 
  width: 100%; 
  overflow: visible`;
container.appendChild(layoutContainer);

// Contenedor del mapa (izquierda)
const mapContainer = document.createElement("div");
mapContainer.style = `
  flex: 0.50; 
  height: 100%;`;
layoutContainer.appendChild(mapContainer);

// Contenedor del gráfico (derecha)
const chartContainer = document.createElement("div");
chartContainer.style = `
  flex: 0.50;
  height: 100%;
  padding: 10px;
  display: flex;
  flex-direction: column;
  overflow: visible;
  margin-left: 10px;
  margin-bottom: 50px;
`;
layoutContainer.appendChild(chartContainer);

// Controles 
// Dropdown de variable
const dropdownContainer = document.createElement("div");
dropdownContainer.style = "margin-bottom: 5px;";
const dropdown = document.createElement("select");
for (const key in variableLabels) {
  const opt = document.createElement("option");
  opt.value = key;
  opt.text = variableLabels[key];
  dropdown.appendChild(opt);
}
dropdown.value = variableSeleccionada;
dropdown.style = "font-size: 14px; padding: 4px; width: 200px;";
dropdownContainer.appendChild(dropdown);
chartContainer.appendChild(dropdownContainer);

// Filtros de fecha
const dateFilter = document.createElement("div");
dateFilter.style = "margin-bottom:5px; font-size:14px;";
const fromDate = document.createElement("input"); fromDate.type = "date";
const toDate   = document.createElement("input"); toDate.type   = "date";
dateFilter.append("Desde ", fromDate, " Hasta ", toDate);
chartContainer.appendChild(dateFilter);

// Filtros de valor
const valueFilter = document.createElement("div");
valueFilter.style = "margin-bottom:10px; font-size:14px;";
const minValue = document.createElement("input"); minValue.type = "number"; minValue.placeholder = "Mín";
const maxValue = document.createElement("input"); maxValue.type = "number"; maxValue.placeholder = "Máx";
minValue.style = maxValue.style = "width:80px; padding:2px; margin-left:5px;";
valueFilter.append("Valor entre", minValue, " y", maxValue);
chartContainer.appendChild(valueFilter);

const smoothControl = document.createElement("div");
smoothControl.style = "font-size: 14px; margin-bottom: 10px; display: flex; align-items: center; gap: 10px;";

// Checkbox de suavizado
const smoothCheckbox = document.createElement("input");
smoothCheckbox.type = "checkbox";
smoothCheckbox.style = "margin-right: 5px;";
const smoothLabel = document.createElement("label");
smoothLabel.appendChild(smoothCheckbox);
smoothLabel.append(" Suavizar el grafico");

smoothCheckbox.addEventListener("change", () => {
  if (lastStationID) {
    buscarEstacion(lastStationID, lastStationName);
  }
});

// Dropdown para tamaño de ventana
const windowSelect = document.createElement("select");
[3, 5, 7, 14, 30].forEach(dias => {
  const opt = document.createElement("option");
  opt.value = dias;
  opt.text = `${dias} días`;
  windowSelect.appendChild(opt);
});
windowSelect.style = "padding: 3px; font-size: 14px;";

windowSelect.addEventListener("change", () => {
  if (lastStationID && smoothCheckbox.checked) {
    buscarEstacion(lastStationID, lastStationName);
  }
});

// Añadir al contenedor
smoothControl.appendChild(smoothLabel);
smoothControl.append(" Ventana:");
smoothControl.appendChild(windowSelect);
chartContainer.appendChild(smoothControl);

// Botón de descarga CSV
const downloadBtn = document.createElement("button");
downloadBtn.textContent = "Descargar CSV";
downloadBtn.style = "align-self: flex-start; margin-bottom:10px; padding:6px 12px;";
chartContainer.appendChild(downloadBtn);

// Título de la estación
const chartTitle = document.createElement("h3");
chartTitle.style = "margin: 5px 0; padding-left: 40px; font-weight: bold; text-align: center;";
chartTitle.textContent = "Selecciona un círculo en el mapa";
chartContainer.appendChild(chartTitle);

// Placeholder inicial
const placeholderDiv = document.createElement("div");
placeholderDiv.style = `
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #888;
  font-size: 1.2em;
`;
placeholderDiv.textContent = "Pase el mouse sobre un punto para ver el gráfico";
chartContainer.appendChild(placeholderDiv);

// Área del plot (oculta al inicio)
const plotArea = document.createElement("div");
plotArea.style = `
  flex: 1;
  width: 100%;
  display: none;
  overflow: visible;
`;
chartContainer.appendChild(plotArea);

// Helpers para alternar placeholder/plot
function mostrarPlaceholder() {
  placeholderDiv.style.display = "flex";
  plotArea.style.display       = "none";
}
function mostrarPlot() {
  placeholderDiv.style.display = "none";
  plotArea.style.display       = "block";
}
mostrarPlaceholder();

// Funcion que maneja el suavizado del grafico
function movingAverage(data, windowSize = 7) {
  const result = [];
  for (let i = 0; i < data.length; i++) {
    const start = Math.max(0, i - windowSize + 1);
    const window = data.slice(start, i + 1);
    const avg = d3.mean(window, d => d.value);
    result.push({ date: data[i].date, value: avg });
  }
  return result;
}

// Inicializar el mapa
const map = L.map(mapContainer).setView([18.2208, -66.5901], 9);
L.tileLayer(
  "https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}",
  { attribution: '&copy; Esri, USGS, NOAA' }
).addTo(map);

// Cargar CSV principal
const df = await FileAttachment("full.csv").csv({ typed: true });

// Función de color por años de datos
function getColor(v) {
  return v > 120 ? '#800026' :
         v > 100 ? '#BD0026' :
         v > 80  ? '#E31A1C' :
         v > 60  ? '#FC4E2A' :
         v > 40  ? '#FD8D3C' :
         v > 20  ? '#FEB24C' :
         v > 15  ? '#FED976' :
                   '#FFEDA0';
}

// Parsear CSV simple
function parseCSV(csvData) {
  return csvData.trim().split("\n").map(line => line.split(","));
}

// Mostrar estaciones en el mapa con popups y hover
function getData(data) {
  const dailyInicio = data.map(d => d['inventory.daily.start']);
  const dailyFinal  = data.map(d => d['inventory.daily.end']);
  const year1       = dailyInicio.map(d => new Date(d).getFullYear());
  const year2       = dailyFinal.map(d => new Date(d).getFullYear());
  const stationID   = data.map(d => d.id);
  const nombreLugar = data.map(d => d['name.en']);
  const pais        = data.map(d => d.country);
  const cantidad    = year2.map((e,i) => e - year1[i]);
  const lat         = data.map(d => parseFloat(d['location.latitude']));
  const lon         = data.map(d => parseFloat(d['location.longitude']));
  const elev        = data.map(d => d['location.elevation']);
  const customdata  = data.map((d,i) => [
    stationID[i], 
    nombreLugar[i], 
    lat[i], 
    lon[i], 
    elev[i], 
    pais[i], 
    d.flag || ""
  ]);

  // Ciclo for que valida las estaciones a graficar
  for (let i = 0; i < lat.length; i++) {
    if (cantidad[i] >= 5 && !isNaN(lat[i]) && !isNaN(lon[i])) {
      const circle = L.circle([lat[i], lon[i]], {
        color: getColor(cantidad[i]),
        fillOpacity: 0.9,
        radius: 7000
      }).addTo(map);

      const popupContent = `
        <b>ID:</b> ${customdata[i][0]}<br>
        <b>Estación:</b> ${customdata[i][1]}<br>
        <b>País:</b> ${customdata[i][5]}<br>
        <b>Latitud:</b> ${customdata[i][2]}<br>
        <b>Longitud:</b> ${customdata[i][3]}<br>
        <b>Elevación:</b> ${customdata[i][4]} m<br>
        <b>Rango de Años:</b> ${year1[i]}–${year2[i]}
      `;
      circle.bindPopup(popupContent);

      circle.on('mouseover', function() {
        this.openPopup();
        buscarEstacion(customdata[i][0], customdata[i][1]);
      });
      circle.on('mouseout', function() {
        this.closePopup();
      });
    }
  }
}

let yLabel = document.createElement("div"); // se actualizará dinámicamente más adelante

// Funcion que dibuja la grafica de linea de las estaciones
async function buscarEstacion(stationID, nombreLugar) {
  lastStationID = stationID;
  lastStationName = nombreLugar;

  const url = `https://cdat.uprh.edu/~eramos/data/${stationID}.csv`;
  try {
    const res = await fetch(url);
    if (!res.ok) throw new Error("Error al obtener CSV");
    const text = await res.text();
    const data = parseCSV(text);

    // Filtros
    const start = fromDate.value ? new Date(fromDate.value) : null;
    const end   = toDate.value   ? new Date(toDate.value)   : null;
    const allDates = data.map(r => new Date(r[0]));
    const idx = variableIndexMap[variableSeleccionada];
    let rawAll = data.map(r => parseFloat(r[idx]));
    if (["tavg","tmin","tmax"].includes(variableSeleccionada))
      rawAll = rawAll.map(v => v * 1.8 + 32);
    const yMin = minValue.value !== "" ? parseFloat(minValue.value) : -Infinity;
    const yMax = maxValue.value !== "" ? parseFloat(maxValue.value) : Infinity;

    currentFilteredData = data.map((r, i) => ({
      date: allDates[i],
      value: rawAll[i]
    })).filter(d =>
      (!start || d.date >= start) &&
      (!end   || d.date <= end) &&
      d.value >= yMin &&
      d.value <= yMax &&
      isFinite(d.value) &&
      !isNaN(d.value)
    );

    chartTitle.innerHTML = `<div>ID: ${stationID}</div><div>${nombreLugar}</div>`;
    mostrarPlot();

    // Preparar yLabel si no ha sido estilizado
    if (!yLabel.hasAttribute("data-initialized")) {
      yLabel.style = `
        writing-mode: vertical-rl;
        transform: rotate(180deg);
        font-size: 14px;
        text-align: center;
        color: currentColor;
        margin-right: 10px;
        padding-right: 5px;
        white-space: nowrap;
      `;
      yLabel.setAttribute("data-initialized", "true");
    }
    yLabel.textContent = variableLabels[variableSeleccionada]; // actualizar texto

    // Crear contenedor para label + plot
    const plotWrapper = document.createElement("div");
    plotWrapper.style = `display: flex; align-items: center;`;
    plotWrapper.appendChild(yLabel);

    const plotAreaInner = document.createElement("div");
    plotAreaInner.style = "flex: 1;";
    plotWrapper.appendChild(plotAreaInner);

    plotArea.innerHTML = "";
    plotArea.appendChild(plotWrapper);

    const windowSize = parseInt(windowSelect.value);
    const datosGraficar = smoothCheckbox.checked
      ? movingAverage(currentFilteredData, windowSize)
      : currentFilteredData;

    // Ancho del plot
    const width = plotArea.getBoundingClientRect().width || 600;

    const plt = Plot.plot({
      y: {
        grid: true,
        label: "",
        labelArrow: null,
        tickFormat: d => `${d.toFixed(1)}`
      },
      x: {
        label: "Fecha",
        labelAnchor: "center",
        labelOffset: 30,
        labelArrow: null,
        nice: true
      },
      marks: [
        Plot.frame(),
        Plot.line(datosGraficar, { 
          x: "date", 
          y: "value",
          tip: true, 
          stroke: "red" 
        })
      ],
      width: width,
      height: 450
    });

    plotAreaInner.appendChild(plt);

  } catch (e) {
    console.error(e);
  }
}


// Inicializar el mapa con estaciones
getData(df);

// Evento de cambio de variable: redibujar si hay estación cargada 
dropdown.addEventListener("change", () => {
  variableSeleccionada = dropdown.value;
  if (lastStationID) {
    buscarEstacion(lastStationID, lastStationName);
  } else {
    mostrarPlaceholder();
  }
});

// Reset placeholder si cambian solo los filtros de fecha/valor
[fromDate, toDate, minValue, maxValue].forEach(el =>
  el.addEventListener("change", mostrarPlaceholder)
);

downloadBtn.addEventListener("click", () => {
  if (!currentFilteredData.length) {
    return alert("No hay datos para descargar.");
  }

  // Cabecera con ID y nombre
  const metadata = `ID: ${lastStationID}, Nombre: ${lastStationName}`;
  const header = "fecha," + variableSeleccionada;
  const rows = currentFilteredData.map(d =>
    `${d.date.toISOString().slice(0, 10)},${d.value.toFixed(1)}`
  );

  const contenido = [metadata, header, ...rows].join("\n");

  const csvBlob = new Blob([contenido], { type: "text/csv" });
  const url = URL.createObjectURL(csvBlob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `${lastStationID}_${variableSeleccionada}.csv`;
  a.click();
  URL.revokeObjectURL(url);
});


```
