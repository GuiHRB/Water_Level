# Explicação Completa do Sistema IoT

> Este documento explica a teoria, arquitetura e funcionamento de cada componente da stack IoT configurada com Docker no WSL2.

---

## Índice

1. [Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
2. [WSL2 — A Base de Tudo](#2-wsl2--a-base-de-tudo)
3. [Docker — O Motor de Containers](#3-docker--o-motor-de-containers)
4. [Conceitos Fundamentais do Docker](#4-conceitos-fundamentais-do-docker)
5. [Portainer — Interface Visual do Docker](#5-portainer--interface-visual-do-docker)
6. [Mosquitto — O Broker MQTT](#6-mosquitto--o-broker-mqtt)
7. [Node-RED — O Motor de Integração](#7-node-red--o-motor-de-integração)
8. [InfluxDB — Banco de Dados de Séries Temporais](#8-influxdb--banco-de-dados-de-séries-temporais)
9. [Grafana — Visualização de Dados](#9-grafana--visualização-de-dados)
10. [Teoria de Redes — Como Tudo se Conecta](#10-teoria-de-redes--como-tudo-se-conecta)
11. [ESP32 — O Dispositivo IoT](#11-esp32--o-dispositivo-iot)
12. [Fluxo Completo de Dados](#12-fluxo-completo-de-dados)
13. [Estrutura de Pastas e Volumes](#13-estrutura-de-pastas-e-volumes)
14. [Tabela de Portas e Serviços](#14-tabela-de-portas-e-serviços)

---

## 1. Visão Geral da Arquitetura

O sistema é composto por camadas que se comunicam em sequência:

```
┌─────────────────────────────────────────────────────┐
│                  DISPOSITIVO FÍSICO                 │
│           ESP32 + Sensor Ultrassônico               │
│         Lê distância → publica via MQTT             │
└───────────────────────┬─────────────────────────────┘
                        │ WiFi → IP do PC (192.168.x.x)
                        ▼
┌─────────────────────────────────────────────────────┐
│                  WINDOWS (HOST)                     │
│     Port Forwarding automático do WSL2              │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│                  WSL2 (Ubuntu)                      │
│              Kernel Linux real                      │
│   ┌─────────────────────────────────────────────┐  │
│   │              DOCKER ENGINE                  │  │
│   │                                             │  │
│   │  [Portainer]  [Mosquitto]  [Node-RED]       │  │
│   │  [InfluxDB ]  [Grafana  ]                   │  │
│   └─────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

Cada serviço roda isolado em seu próprio container Docker, dentro do Ubuntu, que roda dentro do Windows via WSL2.

---

## 2. WSL2 — A Base de Tudo

### O que é

O **Windows Subsystem for Linux 2** é uma camada de compatibilidade da Microsoft que permite rodar um kernel Linux real dentro do Windows, sem dual boot e sem máquina virtual tradicional. Ele usa o Hyper-V para criar uma VM leve e transparente.

### Por que é necessário

O Docker precisa de um kernel Linux para funcionar — containers compartilham o kernel do sistema operacional host. No Windows puro não existe kernel Linux, então o WSL2 fornece esse ambiente.

### Como funciona a rede

O WSL2 tem sua própria interface de rede interna:

```
Windows (192.168.xxx.x)  ←→  WSL2 (172.22.xx.xxx)
```

O Windows faz **port forwarding automático** — qualquer porta exposta no WSL2 fica acessível via `localhost` no Windows. Por isso `localhost:9000` no browser do Windows abre o Portainer que roda dentro do Linux.

### Comandos úteis

```bash
# Ver IPs do WSL
hostname -I

# Iniciar Docker
sudo service docker start

# Verificar se WSL está rodando (no CMD do Windows)
wsl --list --running
```

---

## 3. Docker — O Motor de Containers

### O problema que ele resolve

Sem Docker, instalar um serviço como o Mosquitto significa:
- Instalar dependências manualmente
- Configurar o sistema operacional
- Resolver conflitos de versão com outros softwares
- Repetir todo o processo em cada máquina

Com Docker, você descreve o ambiente uma vez e ele roda idêntico em qualquer lugar.

### O conceito de container

Um container é um processo isolado que tem:
- Seu próprio filesystem
- Sua própria rede virtual
- Suas próprias variáveis de ambiente
- Suas próprias dependências

Mas **compartilha o kernel** do sistema operacional host. Por isso containers são muito mais leves que máquinas virtuais completas.

```
Máquina Virtual:          Container Docker:
┌──────────────┐          ┌──────────────┐
│  App         │          │  App         │
│  Libs        │          │  Libs        │
│  OS completo │          │  (sem OS)    │
│  Hypervisor  │          │  Docker      │
│  Hardware    │          │  Kernel Linux│
└──────────────┘          │  Hardware    │
  ~GB de overhead         └──────────────┘
                            ~MB de overhead
```

### Comparação: manual vs Docker

| Situação | Manual | Docker |
|---|---|---|
| Instalar em outra máquina | Refaz tudo | `docker run` |
| Duas versões do mesmo software | Conflito | Dois containers |
| Algo quebra | Difícil limpar | `docker rm` |
| Equipe com ambientes iguais | "Funciona na minha máquina" | Todos rodam idêntico |
| Subir 5 serviços juntos | 5 instalações manuais | `docker compose up` |

### O flag `--restart=always`

Todos os containers foram criados com `--restart=always`. Isso instrui o Docker a reiniciar o container automaticamente sempre que o Docker daemon iniciar — ou seja, toda vez que você abre o Ubuntu e o Docker sobe, todos os serviços sobem junto automaticamente.

---

## 4. Conceitos Fundamentais do Docker

### Image

O "molde" congelado e imutável. Define o sistema operacional base, dependências instaladas e o comando que o container vai executar. É baixada do Docker Hub (registro público de imagens).

Exemplos de imagens usadas:
- `portainer/portainer-ce:lts`
- `eclipse-mosquitto:latest`
- `nodered/node-red`
- `influxdb:latest`
- `grafana/grafana:latest`

### Container

Uma instância rodando de uma image. É criado a partir da image e pode ser iniciado, parado, reiniciado e deletado. Múltiplos containers podem rodar da mesma image simultaneamente.

### Volume

Armazenamento persistente para containers. Por padrão, tudo dentro de um container é descartado quando ele é deletado. Volumes sobrevivem à exclusão do container.

Existem dois tipos usados neste projeto:

**Volume gerenciado pelo Docker:**
```
/var/lib/docker/volumes/portainer_data/
```
Controlado pelo Docker, não acessível diretamente pelo usuário.

**Bind Mount (pasta local):**
```
/home/guilherme/mosquitto/ → /mosquitto/config/
```
Uma pasta do seu filesystem mapeada para dentro do container. Você acessa e edita os arquivos diretamente.

### Network

Rede virtual criada pelo Docker para containers se comunicarem. A rede padrão `bridge` é usada por todos os containers deste projeto — eles se enxergam internamente e as portas são mapeadas para o host.

### Comandos essenciais

```bash
# Listar containers rodando
docker ps

# Listar todos (inclusive parados)
docker ps -a

# Ver logs de um container
docker logs nome_container
docker logs -f nome_container   # acompanha em tempo real

# Entrar dentro de um container
docker exec -it nome_container sh

# Parar / iniciar / reiniciar
docker stop nome_container
docker start nome_container
docker restart nome_container

# Remover container
docker rm nome_container

# Listar volumes
docker volume ls

# Listar imagens baixadas
docker images

# Ver uso de recursos em tempo real
docker stats
```

---

## 5. Portainer — Interface Visual do Docker

### O que é

Interface web que permite gerenciar o Docker visualmente, sem linha de comando. Ele próprio roda como um container Docker.

### Como funciona

O Portainer recebe acesso ao socket do Docker:

```
-v /var/run/docker.sock:/var/run/docker.sock
```

O arquivo `/var/run/docker.sock` é a interface de comunicação do Docker daemon. Ao montá-lo como volume, o Portainer consegue listar, criar, parar e deletar containers — basicamente tudo que você faria pelo terminal.

### O que você gerencia por ele

- Containers (status, logs, terminal, stats)
- Images (baixadas, em uso)
- Volumes (criados, em uso, unused)
- Redes
- Stacks (Docker Compose)

### Acesso

```
http://localhost:9000
```

---

## 6. Mosquitto — O Broker MQTT

### O que é o protocolo MQTT

MQTT (Message Queuing Telemetry Transport) é um protocolo de comunicação leve, baseado no modelo **publish/subscribe**, projetado para dispositivos com pouca memória e conexões instáveis — exatamente o cenário de IoT.

### Modelo Publish/Subscribe

Diferente do modelo cliente-servidor tradicional (onde cliente fala diretamente com servidor), no MQTT ninguém fala diretamente com ninguém. Tudo passa pelo broker:

```
Publisher (ESP32)        Broker (Mosquitto)        Subscriber (Node-RED)
      │                        │                          │
      │── publica em ─────────▶│                          │
      │   "sensores/distancia" │                          │
      │                        │── repassa para ─────────▶│
      │                        │   quem assinou           │
```

**Publisher** → quem envia mensagens (ESP32)  
**Subscriber** → quem recebe mensagens (Node-RED, mosquitto_sub)  
**Broker** → o intermediário (Mosquitto)  
**Tópico** → o endereço da mensagem (ex: `sensores/distancia`)

### Hierarquia de tópicos

Tópicos são organizados em hierarquia com `/`:

```
sensores/distancia
sensores/temperatura
atuadores/led
atuadores/motor
fabrica/linha1/sensor/temperatura
```

É possível assinar com wildcards:
- `sensores/#` → recebe tudo abaixo de sensores
- `sensores/+/temperatura` → recebe temperatura de qualquer sensor

### Arquivo de configuração explicado

```
allow_anonymous true          # aceita clientes sem usuário/senha
listener 1883 0.0.0.0        # escuta em todos os IPs na porta 1883
persistence true              # salva mensagens em disco
persistence_location /mosquitto/data/  # onde salva
```

### Portas

| Porta | Uso |
|---|---|
| 1883 | MQTT padrão (TCP) |
| 9001 | MQTT sobre WebSocket (para browsers) |

---

## 7. Node-RED — O Motor de Integração

### O que é

Plataforma de programação visual baseada em fluxos. Você conecta "nós" graficamente para criar integrações sem escrever código tradicional.

### Conceito de Flow

Um flow é uma sequência de nós conectados:

```
[MQTT in] ──▶ [function] ──▶ [InfluxDB out]
                  │
                  ▼
              [debug]
```

Cada nó recebe uma mensagem (`msg`), processa e passa para o próximo.

### Por que o status é "healthy"

A imagem do Node-RED tem um **healthcheck** configurado — periodicamente o Docker faz uma requisição HTTP interna para verificar se o serviço está respondendo. O `(healthy)` no `docker ps` indica que está passando.

### Papel na stack IoT

O Node-RED fica no meio da cadeia:
1. Assina tópicos MQTT do Mosquitto
2. Processa os dados (converte unidades, filtra, agrega)
3. Grava no InfluxDB com timestamp

### Acesso

```
http://localhost:1880
```

---

## 8. InfluxDB — Banco de Dados de Séries Temporais

### Por que não usar PostgreSQL ou MySQL?

Bancos relacionais tradicionais armazenam registros genéricos. O InfluxDB é otimizado para um caso específico: **dados com timestamp gerados continuamente**, como leituras de sensores.

Vantagens para IoT:
- Compressão automática de dados temporais
- Queries otimizadas para intervalos de tempo
- Retenção automática de dados (apaga dados antigos automaticamente)
- Alta performance para inserções contínuas

### Conceitos principais

**Bucket** → equivalente ao banco de dados. Define também a política de retenção (ex: guardar dados por 30 dias).

**Measurement** → equivalente à tabela. Agrupa dados do mesmo tipo (ex: `distancia`).

**Field** → o valor medido (ex: `valor: 23.5`). Não é indexado.

**Tag** → metadado indexado (ex: `sensor: "esp32-sala"`). Usado para filtrar.

**Timestamp** → gerado automaticamente em cada inserção.

### Exemplo de dado armazenado

```
measurement: distancia
tags: sensor=esp32-01, local=sala
fields: valor=23.5
timestamp: 2026-06-27T14:30:00Z
```

### Acesso

```
http://localhost:8086
```

---

## 9. Grafana — Visualização de Dados

### O que é

Plataforma de observabilidade e visualização. Conecta em diversas fontes de dados (InfluxDB, PostgreSQL, Prometheus, etc.) e cria dashboards com gráficos, tabelas, gauges e alertas.

### Papel na stack

O Grafana é a camada final — transforma os dados brutos do InfluxDB em visualizações compreensíveis:

- Gráfico de linha mostrando distância ao longo do tempo
- Gauge mostrando o valor atual
- Alerta quando distância ultrapassar um limite

### Acesso

```
http://localhost:3000
```

Login padrão: `admin` / `admin`

---

## 10. Teoria de Redes — Como Tudo se Conecta

### Endereços IP

Todo dispositivo numa rede tem um endereço IP — funciona como o CEP de uma casa.

| Endereço | Significado |
|---|---|
| `127.0.0.1` (localhost) | A própria máquina. Pacotes não saem pela rede. |
| `192.168.x.x` | IP privado na rede local (roteador/switch) |
| `172.22.46.x` | IP interno do WSL2 (invisível para dispositivos externos) |
| IP público | IP do roteador visto pela internet |

### Portas

O IP identifica a máquina, a porta identifica o serviço:

```
192.168.xxx.x : 1883
      │               │
   máquina        serviço (Mosquitto)
```

### O que significa "escutar" numa porta

Quando um serviço inicia, ele abre uma porta e aguarda conexões — isso é o `LISTENING` visto no `netstat`.

O detalhe importante é **em qual endereço ele escuta**:

| Configuração | Comportamento |
|---|---|
| `127.0.0.1:1883` | Só aceita conexões da própria máquina |
| `0.0.0.0:1883` | Aceita conexões de qualquer IP |

Por isso o Mosquitto foi configurado com `listener 1883 0.0.0.0` — sem isso, o ESP32 seria recusado.

### Por que o ESP32 consegue acessar o Mosquitto

O ESP32 conecta no WiFi e recebe um IP da rede local (ex: `192.168.xxx.xx`). O PC tem o IP `192.168.xxx.x`. O caminho real de uma mensagem MQTT é:

```
ESP32 (192.168.xxx.xx)
      │
      │ envia para 192.168.xxx.x:1883
      ▼
Windows (192.168.xxx.x)
      │
      │ Port forwarding (netsh portproxy)
      ▼
WSL2 (172.22.xx.xxx)
      │
      │ porta 1883
      ▼
Container Mosquitto
```

O ESP32 nunca fala diretamente com o WSL — ele fala com o IP do Windows, e o Windows redireciona para dentro do WSL via `netsh portproxy`.

### Por que foi necessário o netsh portproxy

O WSL2 faz port forwarding automático para `localhost`, mas conexões vindas de IPs externos (como o ESP32) às vezes não são roteadas corretamente. O comando:

```cmd
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=1883 connectaddress=172.22.xx.xxx connectport=1883
```

Cria um proxy de porta explícito: qualquer conexão chegando na porta `1883` do Windows é redirecionada para `172.22.xx.xxx:1883` (o WSL).

### Diagrama completo da rede

```
Internet
    │
    ▼
Roteador (192.168.xxx.x)
    │
    ├── PC (192.168.xxx.x) ── cabo ethernet
    │       │
    │       └── WSL2 (172.22.xx.xxx) [rede interna]
    │               │
    │               └── Docker
    │                     ├── Mosquitto :1883
    │                     ├── Node-RED  :1880
    │                     ├── InfluxDB  :8086
    │                     ├── Grafana   :3000
    │                     └── Portainer :9000
    │
    └── ESP32 (192.168.xxx.xx) ── WiFi
```

---

## 11. ESP32 — O Dispositivo IoT

### O que é

Microcontrolador com WiFi e Bluetooth integrados, fabricado pela Espressif. É o dispositivo que lê os sensores e publica dados via MQTT.

### Sensor HC-SR04 — Ultrassônico

O sensor mede distância usando ultrassom:

1. O pino **TRIG** recebe um pulso de 10 microsegundos
2. O sensor emite um burst de ultrassom (40kHz)
3. O sinal reflete no obstáculo e retorna
4. O pino **ECHO** fica em HIGH pelo tempo que o sinal levou

Cálculo da distância:

```
distância = (tempo_echo × velocidade_som) / 2
distância = (duracao × 0.0343) / 2
```

Divide por 2 porque o sinal faz o caminho de ida e volta.

### Biblioteca PubSubClient

É a biblioteca MQTT para Arduino/ESP32. Ela gerencia:
- Conexão TCP com o broker
- Handshake MQTT
- Publicação de mensagens (`publish`)
- Assinatura de tópicos (`subscribe`)
- Keepalive da conexão (`loop`)

### Estados de erro do MQTT (client.state())

| Código | Significado |
|---|---|
| -4 | Timeout de conexão |
| -3 | Conexão perdida |
| -2 | Conexão recusada |
| -1 | Desconectado |
| 0 | Conectado |

O estado `-2` (recusado) que apareceu durante a configuração indicava que o broker estava inacessível — resolvido com o portproxy e firewall.

### Fluxo do código

```
setup()
  ├── Inicializa Serial (115200 baud)
  ├── Configura pinos TRIG e ECHO
  ├── Conecta ao WiFi (loop até conectar)
  └── Conecta ao broker MQTT

loop() — executa continuamente
  ├── client.loop() → mantém conexão MQTT viva
  └── A cada 2 segundos:
        ├── lerDistancia() → pulso no TRIG, mede ECHO
        ├── Valida (0 < distancia < 400 cm)
        ├── Converte float → string (dtostrf)
        └── client.publish() → envia para o broker
```

---

## 12. Fluxo Completo de Dados

```
1. ESP32 lê o sensor HC-SR04
         │
         │ distancia = 23.45 cm
         ▼
2. ESP32 publica via MQTT
         │
         │ tópico: "sensores/distancia"
         │ payload: "23.45"
         │ destino: 192.168.xxx.x:1883
         ▼
3. Windows recebe e redireciona (portproxy)
         │
         ▼
4. Mosquitto recebe a mensagem
         │
         │ verifica quem assinou "sensores/distancia"
         ▼
5. Node-RED recebe (assinou o tópico)
         │
         │ processa: converte string → número
         │ adiciona metadados se necessário
         ▼
6. InfluxDB armazena
         │
         │ measurement: distancia
         │ field: valor = 23.45
         │ timestamp: agora
         ▼
7. Grafana consulta e exibe
         │
         └── gráfico de linha em tempo real
```

---

## 13. Estrutura de Pastas e Volumes

```
/home/guilherme/                    ← home do usuário no Ubuntu
│
├── mosquitto/                      ← arquivos do Mosquitto
│   ├── mosquitto.conf              ← configuração do broker
│   ├── mosquitto_data/             ← bind mount → /mosquitto/data (persistência)
│   ├── mosquitto_log/              ← bind mount → /mosquitto/log (logs)
│   └── config/                     ← bind mount → /mosquitto/config
│
├── nodered/
│   └── nodered_data/               ← bind mount → /data (flows e configurações)
│
└── influxdb/                       ← bind mount → /var/lib/influxdb2 (banco)

/var/lib/docker/volumes/            ← volumes gerenciados pelo Docker
├── portainer_data/                 ← banco do Portainer (usuários, configs)
└── nodered_data/                   ← volume criado mas não utilizado
```

### Diferença entre Bind Mount e Volume gerenciado

**Bind Mount** → você aponta uma pasta do seu filesystem para dentro do container. Você tem controle total, pode editar os arquivos diretamente, fazer backup facilmente.

**Volume gerenciado** → o Docker cria e controla a pasta em `/var/lib/docker/volumes/`. Mais isolado, mais seguro, mas menos acessível diretamente.

---

## 14. Tabela de Portas e Serviços

| Serviço | Porta Externa | Porta Interna | Protocolo | URL de Acesso |
|---|---|---|---|---|
| Portainer | 9000 | 9000 | HTTP | http://localhost:9000 |
| Node-RED | 1880 | 1880 | HTTP | http://localhost:1880 |
| Mosquitto | 1883 | 1883 | MQTT/TCP | — |
| Mosquitto WS | 9001 | 9001 | WebSocket | — |
| InfluxDB | 8086 | 8086 | HTTP | http://localhost:8086 |
| Grafana | 3000 | 3000 | HTTP | http://localhost:3000 |

**Porta Externa** → porta acessível no Windows (via localhost ou IP do PC)  
**Porta Interna** → porta dentro do container Docker
