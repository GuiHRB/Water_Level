# Manual de Desenvolvimento — VScode e PlatformIO IDE
 
> **IDE:** VSCode
> **Extensão:** PlatformIO IDE
 
---
 
## Pré-requisitos
 
- VSCode instalado
- Extensão **PlatformIO IDE** instalada no VSCode

---

## 1. Criar projeto no PlatformIO
 
1. Clica no ícone do PlatformIO na barra lateral
2. `New Project`
3. Board: `Espressif ESP32 Dev Module`
4. Framework: `Arduino`
 
---

## 2. Configurar o platformio.ini
 
Dentro do terminal Ubuntu, execute os comandos abaixo em sequência:
 
```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
lib_deps =
    knolleary/PubSubClient
```
---

## 3. Ligação do HC-SR04 no ESP32
 
| HC-SR04 | ESP32 |
|---------|-------|
| VCC     | 5V    |
| GND     | GND   |
| TRIG    | GPIO 5 |
| ECHO    | GPIO 18 |

---

## 4. Código — src/main.cpp

### 4.1 Bibliotecas

```cpp
#include WiFi.h
#include PubSubClient.h
```
`WiFi.h` é a biblioteca nativa do ESP32 para conexão WiFi. `PubSubClient.h` é a biblioteca responsável pela comunicação MQTT — ela gerencia conexão com o broker, publicação e assinatura de tópicos.

---

### 4.2 Configurações 

```cpp
const char* ssid          = "";
const char* password      = "";
const char* mqtt_broker   = "SEU_IP";
const char* mqtt_username = "";
const char* mqtt_password = "";
const int   mqtt_port     = 1883;
```
Credenciais do WiFi e endereço do broker MQTT. O `mqtt_broker` deve ser o IP do PC na rede local. Como o broker está configurado com `allow_anonymous true`, usuário e senha ficam vazios.

---

### 4.3 Definições e variáveis globais


```cpp
#define TRIG_PIN 5
#define ECHO_PIN 18
#define TOPICO_DISTANCIA "sensores/distancia"

WiFiClient   espClient;
PubSubClient client(espClient);
long         tempoAnterior = 0;
const long   intervalo     = 2000;

```
`TRIG_PIN` e `ECHO_PIN` são os GPIOs conectados ao sensor HC-SR04. `tempoAnterior` e `intervalo` controlam o timer não-bloqueante — a leitura ocorre a cada 2000ms sem usar `delay()` no loop principal.

---

### 4.4 Função `lerDistancia()`

```cpp
float lerDistancia() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  long duracao = pulseIn(ECHO_PIN, HIGH, 30000);
  return (duracao * 0.0343) / 2;
}
```

Aciona o TRIG com um pulso de 10µs, que dispara o sensor ultrassônico. O `pulseIn` mede o tempo em que o ECHO fica em HIGH — esse tempo representa o percurso do som até o obstáculo e de volta. O timeout de 30000µs evita que o código trave se nenhum objeto for detectado.

A distância é calculada por:
```
distancia = (duracao × 0.0343) / 2
```
`0.0343` é a velocidade do som em cm/µs. Divide por 2 porque o sinal percorre ida e volta.

---

### 4.5 Função `connectMQTT()`

```cpp
bool connectMQTT() {
  byte tentativa = 0;
  do {
    String client_id = "ESP32-" + String(WiFi.macAddress());
    if (client.connect(client_id.c_str(), mqtt_username, mqtt_password)) return 1;
    delay(2000);
    tentativa++;
  } while (!client.connected() && tentativa < 5);
  return 0;
}
```

Tenta conectar ao broker até 5 vezes com intervalo de 2 segundos entre cada tentativa. O `client_id` é gerado com o endereço MAC do ESP32 — isso garante um identificador único por dispositivo, evitando conflitos quando há múltiplos ESP32 na mesma rede.

Retorna `1` se conectou com sucesso, `0` se esgotou as tentativas.

---

### 4.6 Função `setup()`

```cpp
void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  client.setServer(mqtt_broker, mqtt_port);
  connectMQTT();
}
```

Executa uma única vez na inicialização. Configura os pinos do sensor, conecta ao WiFi (aguarda em loop até conectar) e estabelece a conexão com o broker MQTT.

---

### 4.7 Função `loop()

```cpp
void loop() {
  if (!client.connected()) connectMQTT();
  client.loop();

  long agora = millis();
  if (agora - tempoAnterior >= intervalo) {
    tempoAnterior = agora;
    float distancia = lerDistancia();
    if (distancia > 0 && distancia < 400) {
      char payload[10];
      dtostrf(distancia, 5, 2, payload);
      client.publish(TOPICO_DISTANCIA, payload);
    }
  }
}
```

Executa continuamente. O `client.loop()` é obrigatório — mantém a conexão MQTT viva e processa mensagens recebidas. 

O timer não-bloqueante (`millis()`) verifica se já passaram 2 segundos desde a última leitura. Quando sim, lê a distância e valida se está dentro do alcance do sensor (0 a 400 cm).

O `dtostrf` converte o `float` para `char[]` com 2 casas decimais antes de publicar — necessário porque o `client.publish` espera uma string, não um número.

---

### 4.8 Upload e teste

1. Conecte o ESP32 via USB
2. Clique na **seta →** na barra inferior do VSCode para fazer o upload

Para validar que o Mosquitto está recebendo, no Ubuntu:
 
```bash
mosquitto_sub -h localhost -p 1883 -t "sensores/distancia"
```
 
Os valores de distância devem aparecer em tempo real.
 
---
