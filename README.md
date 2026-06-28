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
## 🧰 Stack Utilizada
 
| Serviço | Versão | Porta | Descrição |
|---|---|---|---|
| Mosquitto | latest | 1883 / 9001 | Broker MQTT |
| Node-RED | latest | 1880 | Motor de integração |
| InfluxDB | latest | 8086 | Banco de séries temporais |
| Grafana | latest | 3000 | Visualização de dados |
| Portainer CE | lts | 9000 | Gerenciamento Docker |
 
---
 
## 📦 Pré-requisitos
 
- Windows 10/11
- WSL2 com Ubuntu instalado
- Docker instalado no Ubuntu
- VSCode com extensão PlatformIO
- ESP32 Dev Module
- Sensor ultrassônico HC-SR04
  
---

## Documentação

- Manual de explicação : [EXPLICAÇÃO](https://github.com/GuiHRB/Water_Level/blob/main/EXPLICATION.md#explication-section)
- Instalação e Configuração do Ambiente: [MANUAL](https://github.com/GuiHRB/Water_Level/blob/main/MANUAL.md#setup-section)
- Desenvolvimento no VScode: [CODE](https://github.com/GuiHRB/Water_Level/blob/main/DEVELOPER.md#code-section)
- Projeto NODE-RED + InfluxDB + Grafana: [Link Text](#thisll-be-a-helpful-section-about-the-greek-letter-Θ).

---

## 🛠️ Tecnologias
 
- **ESP32** — microcontrolador WiFi
- **HC-SR04** — sensor ultrassônico
- **MQTT** — protocolo de mensagens IoT
- **Docker** — containerização dos serviços
- **WSL2** — ambiente Linux no Windows
- **PlatformIO** — IDE para desenvolvimento embarcado
---
