# LB PR 2

## Руму Максим IПЗ 4.02 Лабораторна-Практична робота № 2

## Тема: Розробка додатку для візуалізації вимірювань радару

## Мета: Розробити додаток, який зчитує дані з емульованої вимірювальної частини радару, наданої у вигляді Docker image, та відображає задетектовані цілі на графіку в полярних координатах.

## 0. Розробити додаток для відображення цілей:

#### Перед початком створювання веб-додатку радару підключаємо контейнер командою `docker pull iperekrestov/university:radar-emulation-service` та командою `docker run --name radar-emulator -p 4000:4000 iperekrestov/university:radar-emulation-service` запускаємо його для получення данних по цілі:

#### Рис. 1 - підключенний контейнер

## 1. Розробити додаток для відображення цілей:

#### Створюємо веб-додаток, який підключається до WebSocket сервера та зчитує дані про задетектовані цілі. Додаток реалізовано у вигляді HTML файлу, який використовує протокол WebSocket для підключення до контейнера та за допомогою бібліотеки Plotly:

#### Рис. 2 - Веб-додаток радару

## 2. Обробка та візуалізація даних:

#### Далі налаштовуюємо додаток таким чином, щоб він обробляв отримані через WebSocket дані та відображав кожну ціль як точку на графіку з координатами (кут, відстань), а також надавав можливість зміни параметрів радару через API запити:

```js
socket.addEventListener('message', (evt) => {
  let msg;
  try {
    msg = JSON.parse(evt.data);
  } catch {
    return;
  }
  handleMessage(msg);
});

function handleMessage(msg) {
  msgCount++;

  scanAngle = msg.scanAngle ?? scanAngle;
  pulseDuration = msg.pulseDuration ?? pulseDuration;

  const responses = msg.echoResponses ?? [];
  const now = Date.now();

  let newCount = 0;
  for (const echo of responses) {
    const dist = timeToDist(echo.time);
    const power = echo.power;

    if (dist <= 0) continue;

    detections.push({
      angle: scanAngle,
      distance: dist,
      power,
      timestamp: now,
    });
    totalDetections++;
    newCount++;

    if (dist > maxObservedDist) maxObservedDist = dist;
  }

  if (detections.length > MAX_DETECTIONS) {
    detections.splice(0, detections.length - MAX_DETECTIONS);
  }

  document.getElementById('infoAngle').textContent = `${scanAngle.toFixed(1)}°`;
  document.getElementById('infoPulse').textContent = `${pulseDuration} µs`;
  document.getElementById('infoTargets').textContent = String(responses.length);
  document.getElementById('infoMaxDist').textContent =
    maxObservedDist > 0 ? `${maxObservedDist.toFixed(1)} км` : '—';
  document.getElementById('infoTotal').textContent = String(totalDetections);

  updateTargetsTable(responses, scanAngle);
}
```

#### Рис. 3 - Оброблення данних

```js
document.getElementById('btnApply').addEventListener('click', async () => {
  const btn = document.getElementById('btnApply');
  const status = document.getElementById('apiStatus');

  const payload = {
    measurementsPerRotation: Number(document.getElementById('paramMPR').value),
    rotationSpeed: Number(document.getElementById('paramRS').value),
    targetSpeed: Number(document.getElementById('paramTS').value),
  };

  if (Object.values(payload).some((v) => isNaN(v) || v < 0)) {
    status.textContent = 'Помилка: невірні значення';
    status.className = 'api-status err';
    return;
  }

  btn.disabled = true;
  btn.textContent = 'Надсилання…';
  status.textContent = '';

  try {
    const res = await fetch(API_URL, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });

    if (res.ok) {
      rotationSpeedRPM = payload.rotationSpeed;
      status.textContent = 'OK';
      status.className = 'api-status ok';
    } else {
      const body = await res.text();
      status.textContent = `Помилка ${res.status}`;
      status.className = 'api-status err';
      console.error('API error:', body);
    }
  } catch (err) {
    status.textContent = "Немає з'єднання";
    status.className = 'api-status err';
    console.error('Fetch error:', err);
  } finally {
    btn.disabled = false;
    btn.textContent = 'Застосувати';
    setTimeout(() => {
      status.textContent = '';
    }, 3000);
  }
});
```

#### Рис. 4 - Зміна параметрів

```js
setInterval(redrawChart, CHART_UPDATE_MS);

function redrawChart() {
  if (!chartReady) return;

  const now = Date.now();
  const rotPeriodMs = (60 / rotationSpeedRPM) * 1000;

  detections = detections.filter((d) => now - d.timestamp < rotPeriodMs);

  const maxR = Math.max(maxObservedDist, 10) * 1.05;
  const sweep = buildSweepTrace(scanAngle, maxR);
  const points = buildTargetsTrace(detections, now);

  Plotly.react('radarChart', [sweep, points], buildLayout(maxR));
}
```

#### Рис. 5 - Оновлення графіку

## 3. Налаштування графіка:

#### Наступним чином відображаємо відстань у кілометрах у радіальній осі та відображаємо різні кольори або стилі точок для відображення різних рівнів потужності сигналів :

#### Рис. 6 - Відображення відстані та потужності

**index.html**

```html
<!doctype html>
<html lang="uk">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Радарний дисплей</title>
    <link rel="stylesheet" href="style.css" />
  </head>
  <body>
    <header class="header">
      <h1>Радарний дисплей</h1>
      <div class="header-right">
        <div class="status-badge" id="statusBadge">
          <span class="status-dot"></span>
          <span id="statusText">Відключено</span>
        </div>
        <button class="btn-reconnect" id="btnReconnect">
          Перепідключитись
        </button>
      </div>
    </header>

    <main class="main">
      <section class="chart-section">
        <div id="radarChart"></div>
      </section>

      <aside class="sidebar">
        <div class="panel">
          <div class="panel-title">Інформація про сканування</div>
          <div class="info-grid">
            <div class="info-item">
              <span class="info-label">Кут</span>
              <span class="info-value" id="infoAngle">—°</span>
            </div>
            <div class="info-item">
              <span class="info-label">Імпульс</span>
              <span class="info-value" id="infoPulse">— µs</span>
            </div>
            <div class="info-item">
              <span class="info-label">Цілей</span>
              <span class="info-value highlight" id="infoTargets">0</span>
            </div>
            <div class="info-item">
              <span class="info-label">Макс. відстань</span>
              <span class="info-value" id="infoMaxDist">— км</span>
            </div>
            <div class="info-item">
              <span class="info-label">Повід. / сек</span>
              <span class="info-value" id="infoMsgRate">0</span>
            </div>
            <div class="info-item">
              <span class="info-label">Всього</span>
              <span class="info-value" id="infoTotal">0</span>
            </div>
          </div>
        </div>

        <div class="panel">
          <div class="panel-title">Параметри</div>
          <div class="params-form">
            <div class="param-row">
              <label class="param-label" for="paramMPR">
                Вимірювань на оберт
                <span class="param-unit">разів</span>
              </label>
              <div class="param-input-row">
                <input
                  type="range"
                  class="param-slider"
                  id="sliderMPR"
                  min="10"
                  max="1000"
                  step="10"
                  value="360"
                />
                <input
                  type="number"
                  class="param-number"
                  id="paramMPR"
                  min="10"
                  max="1000"
                  step="10"
                  value="360"
                />
              </div>
            </div>

            <div class="param-row">
              <label class="param-label" for="paramRS">
                Швидкість обертання
                <span class="param-unit">об/хв</span>
              </label>
              <div class="param-input-row">
                <input
                  type="range"
                  class="param-slider"
                  id="sliderRS"
                  min="1"
                  max="60"
                  step="1"
                  value="10"
                />
                <input
                  type="number"
                  class="param-number"
                  id="paramRS"
                  min="1"
                  max="60"
                  step="1"
                  value="10"
                />
              </div>
            </div>

            <div class="param-row">
              <label class="param-label" for="paramTS">
                Швидкість цілей
                <span class="param-unit">км/год</span>
              </label>
              <div class="param-input-row">
                <input
                  type="range"
                  class="param-slider"
                  id="sliderTS"
                  min="0"
                  max="2000"
                  step="10"
                  value="100"
                />
                <input
                  type="number"
                  class="param-number"
                  id="paramTS"
                  min="0"
                  max="2000"
                  step="10"
                  value="100"
                />
              </div>
            </div>

            <div class="btn-row">
              <button class="btn-apply" id="btnApply">Застосувати</button>
              <span class="api-status" id="apiStatus"></span>
            </div>
          </div>
        </div>

        <div class="panel">
          <div class="panel-title">Потужність сигналу</div>
          <div class="legend">
            <div class="legend-bar"></div>
            <div class="legend-bar-labels">
              <span>0.0</span>
              <span>0.5</span>
              <span>1.0</span>
            </div>
            <div class="legend-items">
              <div class="legend-item">
                <span class="legend-dot" style="background: #74c7ec"></span>
                <span>Низька (0.0 – 0.3)</span>
              </div>
              <div class="legend-item">
                <span class="legend-dot" style="background: #eed49f"></span>
                <span>Середня (0.3 – 0.7)</span>
              </div>
              <div class="legend-item">
                <span class="legend-dot" style="background: #ed8796"></span>
                <span>Висока (0.7 – 1.0)</span>
              </div>
            </div>
          </div>
        </div>

        <div class="panel panel-table">
          <div class="panel-title">Останні цілі</div>
          <table class="targets-table">
            <thead>
              <tr>
                <th>Кут</th>
                <th>Відстань</th>
                <th>Потужність</th>
              </tr>
            </thead>
            <tbody id="targetsBody">
              <tr>
                <td colspan="3" class="no-data">Очікування даних…</td>
              </tr>
            </tbody>
          </table>
        </div>
      </aside>
    </main>
    <script src="https://cdn.plot.ly/plotly-2.35.2.min.js"></script>
    <script src="script.js"></script>
  </body>
</html>
```

**style.css**

```css
*,
*::before,
*::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

:root {
  --bg: #1e2030;
  --bg-panel: #24273a;
  --bg-input: #2a2d3d;
  --border: #363a4f;
  --text: #cad3f5;
  --text-muted: #6c7086;
  --accent: #8aadf4;
  --accent-dim: rgba(138, 173, 244, 0.1);
  --green: #a6da95;
  --red: #ed8796;
  --yellow: #eed49f;
  --font-ui: system-ui, -apple-system, sans-serif;
  --font-mono: 'Courier New', monospace;
}

html,
body {
  height: 100%;
  background: var(--bg);
  color: var(--text);
  font-family: var(--font-ui);
  font-size: 13px;
  overflow: hidden;
}

.header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 20px;
  background: var(--bg-panel);
  border-bottom: 1px solid var(--border);
  height: 48px;
  flex-shrink: 0;
}

.header h1 {
  font-size: 15px;
  font-weight: 600;
  color: var(--text);
  letter-spacing: 0.3px;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 10px;
}

.status-badge {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: var(--text-muted);
}

.status-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--text-muted);
  flex-shrink: 0;
  transition: background 0.3s;
}

.status-badge.connected .status-dot {
  background: var(--green);
}
.status-badge.disconnected .status-dot {
  background: var(--red);
}
.status-badge.connecting .status-dot {
  background: var(--yellow);
  animation: blink 0.9s step-start infinite;
}

.status-badge.connected #statusText {
  color: var(--green);
}
.status-badge.disconnected #statusText {
  color: var(--red);
}
.status-badge.connecting #statusText {
  color: var(--yellow);
}

@keyframes blink {
  50% {
    opacity: 0;
  }
}

.btn-reconnect {
  background: transparent;
  border: 1px solid var(--border);
  color: var(--text-muted);
  font-family: var(--font-ui);
  font-size: 12px;
  padding: 4px 10px;
  border-radius: 4px;
  cursor: pointer;
  transition:
    border-color 0.15s,
    color 0.15s;
}

.btn-reconnect:hover {
  border-color: var(--accent);
  color: var(--accent);
}

.main {
  display: flex;
  height: calc(100vh - 48px);
  overflow: hidden;
}

.chart-section {
  flex: 1;
  min-width: 0;
  padding: 16px;
}

#radarChart {
  width: 100%;
  height: 100%;
}

.sidebar {
  width: 276px;
  flex-shrink: 0;
  border-left: 1px solid var(--border);
  background: var(--bg-panel);
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  overflow-x: hidden;
}

.sidebar::-webkit-scrollbar {
  width: 3px;
}
.sidebar::-webkit-scrollbar-thumb {
  background: var(--border);
}

.panel {
  padding: 14px 16px;
  border-bottom: 1px solid var(--border);
}

.panel-title {
  font-size: 11px;
  font-weight: 600;
  color: var(--text-muted);
  letter-spacing: 0.5px;
  text-transform: uppercase;
  margin-bottom: 10px;
}

.info-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px;
}

.info-item {
  background: var(--bg-input);
  border-radius: 4px;
  padding: 7px 9px;
}

.info-label {
  display: block;
  font-size: 10px;
  color: var(--text-muted);
  margin-bottom: 2px;
}

.info-value {
  display: block;
  font-size: 15px;
  font-weight: 600;
  font-family: var(--font-mono);
  color: var(--text);
}

.info-value.highlight {
  color: var(--accent);
}

.params-form {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.param-row {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.param-label {
  font-size: 12px;
  color: var(--text);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.param-unit {
  font-size: 11px;
  color: var(--text-muted);
}

.param-input-row {
  display: flex;
  align-items: center;
  gap: 8px;
}

.param-slider {
  flex: 1;
  -webkit-appearance: none;
  height: 3px;
  background: var(--border);
  outline: none;
  border-radius: 2px;
  cursor: pointer;
}

.param-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 13px;
  height: 13px;
  background: var(--accent);
  border-radius: 50%;
  cursor: pointer;
}

.param-slider::-moz-range-thumb {
  width: 13px;
  height: 13px;
  background: var(--accent);
  border: none;
  border-radius: 50%;
  cursor: pointer;
}

.param-number {
  width: 68px;
  background: var(--bg-input);
  border: 1px solid var(--border);
  border-radius: 4px;
  color: var(--text);
  font-family: var(--font-mono);
  font-size: 12px;
  padding: 4px 6px;
  text-align: right;
  outline: none;
  transition: border-color 0.15s;
}

.param-number:focus {
  border-color: var(--accent);
}

.param-number::-webkit-inner-spin-button,
.param-number::-webkit-outer-spin-button {
  -webkit-appearance: none;
}
.param-number[type='number'] {
  -moz-appearance: textfield;
}

.btn-row {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-top: 2px;
}

.btn-apply {
  flex: 1;
  background: var(--accent-dim);
  border: 1px solid var(--accent);
  border-radius: 4px;
  color: var(--accent);
  font-family: var(--font-ui);
  font-size: 12px;
  font-weight: 500;
  padding: 6px 12px;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-apply:hover {
  background: rgba(138, 173, 244, 0.18);
}
.btn-apply:active {
  background: rgba(138, 173, 244, 0.25);
}
.btn-apply:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.api-status {
  font-size: 11px;
  min-width: 40px;
  text-align: right;
}

.api-status.ok {
  color: var(--green);
}
.api-status.err {
  color: var(--red);
}

.legend {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.legend-bar {
  height: 8px;
  width: 100%;
  border-radius: 3px;
  background: linear-gradient(to right, #74c7ec, #eed49f, #ed8796);
}

.legend-bar-labels {
  display: flex;
  justify-content: space-between;
  font-size: 10px;
  color: var(--text-muted);
  margin-top: 3px;
}

.legend-items {
  display: flex;
  flex-direction: column;
  gap: 5px;
  margin-top: 2px;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  color: var(--text-muted);
}

.legend-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
}

.panel-table {
  flex: 1;
}

.targets-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 12px;
  font-family: var(--font-mono);
}

.targets-table th {
  text-align: left;
  color: var(--text-muted);
  font-size: 10px;
  font-family: var(--font-ui);
  font-weight: 500;
  padding: 0 6px 6px;
}

.targets-table td {
  padding: 4px 6px;
  border-top: 1px solid var(--border);
}

.targets-table tr:hover td {
  background: var(--accent-dim);
}

.no-data {
  text-align: center;
  color: var(--text-muted);
  padding: 14px 6px !important;
  font-family: var(--font-ui);
  font-style: italic;
  border-top: none !important;
}

.power-low {
  color: #74c7ec;
}
.power-medium {
  color: #eed49f;
}
.power-high {
  color: #ed8796;
}
```

**script.js**

```js
const WS_URL = 'ws://localhost:4000';
const API_URL = 'http://localhost:4000/config';
const C = 299_792_458;
const MAX_DETECTIONS = 2000;
const CHART_UPDATE_MS = 80;
const TRAIL_DEGREES = 25;
const RECONNECT_DELAY = 3000;

let socket = null;
let scanAngle = 0;
let pulseDuration = 1;
let detections = [];
let rotationSpeedRPM = 10;
let msgCount = 0;
let totalDetections = 0;
let maxObservedDist = 0;
let reconnectTimer = null;
let chartReady = false;

function timeToDist(timeSec) {
  return (timeSec * C) / 2 / 1000;
}

function powerToRgba(power, alpha = 1) {
  const p = Math.max(0, Math.min(1, power));
  let r, g, b;
  if (p < 0.5) {
    const t = p * 2;
    r = Math.round(0x74 + (0xee - 0x74) * t);
    g = Math.round(0xc7 + (0xd4 - 0xc7) * t);
    b = Math.round(0xec + (0x9f - 0xec) * t);
  } else {
    const t = (p - 0.5) * 2;
    r = Math.round(0xee + (0xed - 0xee) * t);
    g = Math.round(0xd4 + (0x87 - 0xd4) * t);
    b = Math.round(0x9f + (0x96 - 0x9f) * t);
  }
  return `rgba(${r},${g},${b},${alpha})`;
}

function powerClass(power) {
  if (power < 0.3) return 'power-low';
  if (power < 0.7) return 'power-medium';
  return 'power-high';
}

function buildLayout(maxR) {
  return {
    polar: {
      bgcolor: '#181a2a',
      angularaxis: {
        direction: 'clockwise',
        rotation: 90,
        tickfont: {
          color: '#6c7086',
          size: 10,
          family: 'system-ui, sans-serif',
        },
        gridcolor: 'rgba(138,173,244,0.1)',
        linecolor: 'rgba(138,173,244,0.2)',
        tickmode: 'array',
        tickvals: [0, 30, 60, 90, 120, 150, 180, 210, 240, 270, 300, 330],
        ticktext: [
          'N',
          '30°',
          '60°',
          'E',
          '120°',
          '150°',
          'S',
          '210°',
          '240°',
          'W',
          '300°',
          '330°',
        ],
      },
      radialaxis: {
        tickfont: {
          color: '#6c7086',
          size: 9,
          family: 'system-ui, sans-serif',
        },
        gridcolor: 'rgba(138,173,244,0.08)',
        linecolor: 'rgba(138,173,244,0.15)',
        ticksuffix: ' км',
        range: [0, maxR],
        showticklabels: true,
      },
    },
    paper_bgcolor: '#1e2030',
    margin: { t: 10, b: 10, l: 10, r: 10 },
    showlegend: false,
    hoverlabel: {
      bgcolor: '#24273a',
      bordercolor: '#8aadf4',
      font: { color: '#cad3f5', family: 'system-ui, sans-serif', size: 12 },
    },
  };
}

function buildSweepTrace(angle, maxR) {
  const steps = 30;
  const r = [0];
  const theta = [angle];

  for (let i = 0; i <= steps; i++) {
    const a = (angle - TRAIL_DEGREES * (i / steps) + 360) % 360;
    r.push(maxR);
    theta.push(a);
  }
  r.push(0);
  theta.push(angle);

  return {
    type: 'scatterpolar',
    r,
    theta,
    mode: 'lines',
    fill: 'toself',
    fillcolor: 'rgba(138,173,244,0.06)',
    line: { color: 'rgba(138,173,244,0.6)', width: 2 },
    hoverinfo: 'skip',
    showlegend: false,
  };
}

function buildTargetsTrace(dets, now) {
  if (dets.length === 0) {
    return {
      type: 'scatterpolar',
      r: [],
      theta: [],
      mode: 'markers',
      showlegend: false,
    };
  }

  const rotPeriodMs = (60 / rotationSpeedRPM) * 1000;

  const r = [];
  const theta = [];
  const colors = [];
  const sizes = [];
  const texts = [];

  for (const d of dets) {
    const age = Math.max(0, now - d.timestamp);
    const alpha = Math.max(0.05, 1 - age / rotPeriodMs);

    r.push(d.distance);
    theta.push(d.angle);
    colors.push(powerToRgba(d.power, alpha));
    sizes.push(5 + d.power * 9);
    texts.push(
      `Кут: ${d.angle.toFixed(1)}°` +
        `<br>Відстань: ${d.distance.toFixed(2)} км` +
        `<br>Потужність: ${(d.power * 100).toFixed(1)}%` +
        `<br>Вік: ${(age / 1000).toFixed(1)} с`,
    );
  }

  return {
    type: 'scatterpolar',
    r,
    theta,
    mode: 'markers',
    marker: { color: colors, size: sizes, symbol: 'circle' },
    text: texts,
    hoverinfo: 'text',
    showlegend: false,
  };
}

function initChart() {
  const maxR = Math.max(maxObservedDist, 10);
  const data = [buildSweepTrace(0, maxR), buildTargetsTrace([], Date.now())];
  const layout = buildLayout(maxR);
  const config = {
    displayModeBar: false,
    responsive: true,
    scrollZoom: false,
  };
  Plotly.newPlot('radarChart', data, layout, config);
  chartReady = true;
}

function redrawChart() {
  if (!chartReady) return;

  const now = Date.now();
  const rotPeriodMs = (60 / rotationSpeedRPM) * 1000;

  detections = detections.filter((d) => now - d.timestamp < rotPeriodMs);

  const maxR = Math.max(maxObservedDist, 10) * 1.05;
  const sweep = buildSweepTrace(scanAngle, maxR);
  const points = buildTargetsTrace(detections, now);

  Plotly.react('radarChart', [sweep, points], buildLayout(maxR));
}

function connect() {
  if (socket && socket.readyState < 2) return;

  setStatus('connecting');
  socket = new WebSocket(WS_URL);

  socket.addEventListener('open', () => {
    setStatus('connected');
    if (reconnectTimer) {
      clearTimeout(reconnectTimer);
      reconnectTimer = null;
    }
    if (!chartReady) initChart();
  });

  socket.addEventListener('message', (evt) => {
    let msg;
    try {
      msg = JSON.parse(evt.data);
    } catch {
      return;
    }
    handleMessage(msg);
  });

  socket.addEventListener('close', () => {
    setStatus('disconnected');
    reconnectTimer = setTimeout(connect, RECONNECT_DELAY);
  });

  socket.addEventListener('error', () => {
    socket.close();
  });
}

function handleMessage(msg) {
  msgCount++;

  scanAngle = msg.scanAngle ?? scanAngle;
  pulseDuration = msg.pulseDuration ?? pulseDuration;

  const responses = msg.echoResponses ?? [];
  const now = Date.now();

  let newCount = 0;
  for (const echo of responses) {
    const dist = timeToDist(echo.time);
    const power = echo.power;

    if (dist <= 0) continue;

    detections.push({
      angle: scanAngle,
      distance: dist,
      power,
      timestamp: now,
    });
    totalDetections++;
    newCount++;

    if (dist > maxObservedDist) maxObservedDist = dist;
  }

  if (detections.length > MAX_DETECTIONS) {
    detections.splice(0, detections.length - MAX_DETECTIONS);
  }

  document.getElementById('infoAngle').textContent = `${scanAngle.toFixed(1)}°`;
  document.getElementById('infoPulse').textContent = `${pulseDuration} µs`;
  document.getElementById('infoTargets').textContent = String(responses.length);
  document.getElementById('infoMaxDist').textContent =
    maxObservedDist > 0 ? `${maxObservedDist.toFixed(1)} км` : '—';
  document.getElementById('infoTotal').textContent = String(totalDetections);

  updateTargetsTable(responses, scanAngle);
}

function updateTargetsTable(responses, angle) {
  const tbody = document.getElementById('targetsBody');
  if (responses.length === 0) return;

  const rows = responses
    .slice(0, 8)
    .map((echo) => {
      const dist = timeToDist(echo.time);
      const power = echo.power;
      const cls = powerClass(power);
      return `<tr>
      <td>${angle.toFixed(1)}°</td>
      <td>${dist.toFixed(2)} км</td>
      <td class="${cls}">${(power * 100).toFixed(1)}%</td>
    </tr>`;
    })
    .join('');

  tbody.innerHTML = rows;
}

function setStatus(state) {
  const badge = document.getElementById('statusBadge');
  const text = document.getElementById('statusText');
  badge.className = `status-badge ${state}`;
  const labels = {
    connected: 'Підключено',
    disconnected: 'Відключено',
    connecting: 'Підключення…',
  };
  text.textContent = labels[state] ?? state.toUpperCase();
}

setInterval(() => {
  document.getElementById('infoMsgRate').textContent = String(msgCount);
  msgCount = 0;
}, 1000);

setInterval(redrawChart, CHART_UPDATE_MS);

function syncInputs(sliderId, numberId) {
  const slider = document.getElementById(sliderId);
  const number = document.getElementById(numberId);

  slider.addEventListener('input', () => {
    number.value = slider.value;
    if (numberId === 'paramRS') rotationSpeedRPM = Number(slider.value);
  });

  number.addEventListener('input', () => {
    const v = Number(number.value);
    if (!isNaN(v)) {
      slider.value = v;
      if (numberId === 'paramRS') rotationSpeedRPM = v;
    }
  });
}

syncInputs('sliderMPR', 'paramMPR');
syncInputs('sliderRS', 'paramRS');
syncInputs('sliderTS', 'paramTS');

document.getElementById('btnApply').addEventListener('click', async () => {
  const btn = document.getElementById('btnApply');
  const status = document.getElementById('apiStatus');

  const payload = {
    measurementsPerRotation: Number(document.getElementById('paramMPR').value),
    rotationSpeed: Number(document.getElementById('paramRS').value),
    targetSpeed: Number(document.getElementById('paramTS').value),
  };

  if (Object.values(payload).some((v) => isNaN(v) || v < 0)) {
    status.textContent = 'Помилка: невірні значення';
    status.className = 'api-status err';
    return;
  }

  btn.disabled = true;
  btn.textContent = 'Надсилання…';
  status.textContent = '';

  try {
    const res = await fetch(API_URL, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });

    if (res.ok) {
      rotationSpeedRPM = payload.rotationSpeed;
      status.textContent = 'OK';
      status.className = 'api-status ok';
    } else {
      const body = await res.text();
      status.textContent = `Помилка ${res.status}`;
      status.className = 'api-status err';
      console.error('API error:', body);
    }
  } catch (err) {
    status.textContent = "Немає з'єднання";
    status.className = 'api-status err';
    console.error('Fetch error:', err);
  } finally {
    btn.disabled = false;
    btn.textContent = 'Застосувати';
    setTimeout(() => {
      status.textContent = '';
    }, 3000);
  }
});

document.getElementById('btnReconnect').addEventListener('click', () => {
  if (socket) {
    socket.close();
  }
  if (reconnectTimer) {
    clearTimeout(reconnectTimer);
    reconnectTimer = null;
  }
  connect();
});

initChart();
connect();
```

Висновок: Протягом виконання лабораторно-практичної роботи я розробив додаток, який зчитує дані з емульованої вимірювальної частини радару, наданої у вигляді Docker image, та відображає задетектовані цілі на графіку в полярних координатах.
