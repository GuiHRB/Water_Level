# 💧 Water Level Monitor — IoT Stack com ESP32 + MQTT + Docker
 
Sistema de monitoramento de distância/nível utilizando sensor ultrassônico HC-SR04, ESP32 e uma stack IoT completa rodando em containers Docker no WSL2.
 
---
 
## 🏗️ Arquitetura
 
```
ESP32 + HC-SR04
      │
      │ MQTT (WiFi)
      ▼
Mosquitto (Broker MQTT)
      │
      │ Subscribe
      ▼
Node-RED (Processamento)
      │
      │ Escrita
      ▼
InfluxDB (Banco de Séries Temporais)
      │
      │ Query
      ▼
Grafana (Dashboard)
```
 
---
