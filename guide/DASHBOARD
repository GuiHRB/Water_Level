# Manual de Implementação — Node-RED → InfluxDB → Grafana
 
> Esta etapa cobre a integração dos dados do sensor com o banco de dados e a visualização em dashboard.
 
---
 
## Índice
 
1. [Instalar o node do InfluxDB no Node-RED](#1-instalar-o-node-do-influxdb-no-node-red)
2. [Configurar o Flow no Node-RED](#2-configurar-o-flow-no-node-red)
3. [Configurar o nó MQTT in](#3-configurar-o-nó-mqtt-in)
4. [Configurar o nó function](#4-configurar-o-nó-function)
5. [Configurar o nó InfluxDB out](#5-configurar-o-nó-influxdb-out)
6. [Verificar dados no InfluxDB](#6-verificar-dados-no-influxdb)
7. [Configurar o Grafana](#7-configurar-o-grafana)
8. [Criar o Dashboard](#8-criar-o-dashboard)
---

## 1. Instalar o node do InfluxDB no Node-RED
 
Acesse `http://localhost:1880`, clique no menu superior direito (≡) e vá em:
 
**Manage palette → Install**
 
Pesquise por:
 
```
node-red-contrib-influxdb
```
 
Instale e aguarde o Node-RED reiniciar.
 
---

## 2. Configurar o Flow no Node-RED
 
O flow segue a estrutura:
 
```
[mqtt in] ──▶ [debug]
     │
     └──▶ [function] ──▶ [influxdb out]
```
 
- **mqtt in** → recebe os dados do sensor via MQTT
- **debug** → monitora os dados chegando em tempo real
- **function** → transforma e calcula os dados antes de gravar
- **influxdb out** → grava no banco de dados

---

## 3. Configurar o nó MQTT in
 
Duplo clique no nó **mqtt in** e preencha:
 
| Campo | Valor |
|---|---|
| Server | `IP_DO_WSL:1883` |
| Topic | `sensores/distancia` |
| Output | `auto-detect` |
| Name | `Dados` |
 
Na configuração do servidor:
 
| Campo | Valor |
|---|---|
| Server | `IP_DO_WSL` |
| Port | `1883` |
| Protocol | `MQTT V3.1.1` |
 
Clique **Update** → **Done**.
 
---
 
## 4. Configurar o nó function
 
O nó `function` recebe a distância bruta do sensor e calcula três métricas derivadas antes de gravar no InfluxDB.
 
Duplo clique no nó **function** e cole o seguinte código na aba **On Message**:
 
```javascript
msg.payload = [{
    coluna: parseFloat(msg.payload),
    nivel: ((16.38 - parseFloat(msg.payload)) / 16.38) * 100,
    volume: 3.14 * (16.38 - parseFloat(msg.payload)) * 3.25**2
}];
 
return msg;
```

## O que cada campo calcula
 
**`coluna`** → distância bruta lida pelo sensor em cm. É o valor direto do HC-SR04.
 
**`nivel`** → percentual de preenchimento do reservatório. Considera `16.38 cm` como a altura total do reservatório vazio (distância máxima do sensor até o fundo). Quanto menor a distância lida, maior o nível:
 
```
nivel = ((altura_total - distancia_lida) / altura_total) × 100
```
 
**`volume`** → volume de água estimado em cm³, usando a fórmula do cilindro com raio de `3.25 cm`:
 
```
volume = π × altura_agua × r²
altura_agua = altura_total - distancia_lida
```
 
> Ajuste os valores `16.38` (altura do reservatório) e `3.25` (raio) conforme as dimensões reais do seu reservatório.
 
Clique **Done**.
 
---

## 5. Configurar o nó InfluxDB out
 
Duplo clique no nó **influxdb out**.
 
### Configurar a conexão (ícone de lápis ao lado de Server)
 
| Campo | Valor |
|---|---|
| Version | `2.0` |
| URL | `http://IP_DO_WSL:8086` |
| Token | seu token do InfluxDB |
 
Clique **Update**.
 
### Configurar o nó
 
| Campo | Valor |
|---|---|
| Organization | `sua_organização` |
| Bucket | `nodered` |
| Measurement | `distancia` |
| Time Precision | `Milliseconds (ms)` |
 
Clique **Done** → **Deploy**.
 
---
 
 ## 6. Verificar dados no InfluxDB
 
Acesse `http://localhost:8086` → **Data Explorer**.
 
Selecione:
- **FROM:** `nodered`
- **Filter:** `_measurement` → `distancia`
Selecione os fields `coluna`, `nivel` e `volume` para confirmar que os dados estão chegando.
 
Ou use o **Script Editor** com a query Flux:
 
```flux
from(bucket: "nodered")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "distancia")
  |> filter(fn: (r) => r._field == "coluna")
```
 
Clique **Submit** — os pontos devem aparecer no gráfico.
 
---

## 7. Configurar o Grafana
 
Acesse `http://localhost:3000` (login: `admin` / `admin`).
 
### Adicionar InfluxDB como fonte de dados
 
**Connections → Data Sources → Add data source → InfluxDB**
 
| Campo | Valor |
|---|---|
| Query Language | `Flux` |
| URL | `http://IP_DO_WSL:8086` |
| Organization | `sua_organização` |
| Token | seu token do InfluxDB |
| Default Bucket | `nodered` |
 
Clique **Save & Test** — deve aparecer `datasource is working`.
 
---
 
## 8. Criar o Dashboard
 
**Dashboards → New → New Dashboard → Add visualization**
 
Selecione o InfluxDB como fonte de dados.
 
---
 
### Painel 1 — Coluna de água (Gauge)
 
Visualization: `Gauge`  
Title: `Coluna de água`
 
Query Flux:
 
```flux
from(bucket: "nodered")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "distancia")
  |> filter(fn: (r) => r._field == "coluna")
```
 
Configurações do Gauge:
- Unit: `Centimeter (cm)`
- Min: `0`
- Max: `16.38`
---
 
### Painel 2 — Nível do reservatório (Gauge %)
 
Visualization: `Gauge`  
Title: `Nível do reservatório`
 
Query Flux:
 
```flux
from(bucket: "nodered")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "distancia")
  |> filter(fn: (r) => r._field == "nivel")
```
 
Configurações do Gauge:
- Unit: `Percent (0-100)`
- Min: `0`
- Max: `100`
---
 
### Painel 3 — Volume de água (Time series)
 
Visualization: `Time series`  
Title: `Volume de água`
 
Query Flux:
 
```flux
from(bucket: "nodered")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "distancia")
  |> filter(fn: (r) => r._field == "volume")
```
 
Configurações:
- Unit: `cm³`
---
 
### Painel 4 — Nível do reservatório histórico (Time series)
 
Visualization: `Time series`  
Title: `Nível do reservatório`
 
Query Flux:
 
```flux
from(bucket: "nodered")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "distancia")
  |> filter(fn: (r) => r._field == "nivel")
```
 
Configurações:
- Unit: `Percent (0-100)`
---
 
### Salvar o dashboard
 
Clique em **Save** no canto superior direito → defina o nome `Water_Level` → **Save**.
 
Configure o auto-refresh clicando no ícone de relógio no topo:
- Refresh: `5s`
- Time range: `Last 5 minutes`
---
 
## Resultado Final
 
```
ESP32 + HC-SR04
      │ distancia bruta (cm)
      ▼
Mosquitto
      │ tópico: sensores/distancia
      ▼
Node-RED
      │ calcula coluna, nivel e volume
      ▼
InfluxDB (bucket: nodered, measurement: distancia)
      │ fields: coluna | nivel | volume
      ▼
Grafana Dashboard (Water_Level)
      ├── Gauge: Coluna de água (cm)
      ├── Gauge: Nível do reservatório (%)
      ├── Time series: Volume de água (cm³)
      └── Time series: Nível histórico (%)
```
 
 
