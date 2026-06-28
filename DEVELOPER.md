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

