# Manual de Desenvolvimento — VScode e PlatformIO IDE
 
> **IDE:** VSCode
> **Extensão:** PlatformIO IDE
 
---
 
## Índice
 
1. [Instalação do WSL2 e Ubuntu](#1-instalação-do-wsl2-e-ubuntu)
2. [Instalação do Docker](#2-instalação-do-docker)
3. [Configuração do Docker para iniciar automaticamente](#3-configuração-do-docker-para-iniciar-automaticamente)
4. [Instalação do Portainer](#4-instalação-do-portainer)
5. [Instalação do Mosquitto](#5-instalação-do-mosquitto)
6. [Instalação do Node-RED](#6-instalação-do-node-red)
7. [Instalação do InfluxDB](#7-instalação-do-influxdb)
8. [Instalação do Grafana](#8-instalação-do-grafana)
9. [Configuração de Rede — ESP32 acessar o broker](#9-configuração-de-rede--esp32-acessar-o-broker)

---

## 1. Instalação do WSL2 e Ubuntu
 
Abra o **PowerShell como administrador** e execute:
 
```powershell
wsl --install
```
 
Reinicie o PC quando solicitado. Após reiniciar, o Ubuntu abrirá automaticamente pedindo para criar um usuário e senha.
 
Para verificar se o WSL2 está ativo:
 
```powershell
wsl --list --verbose
```

Caso o Ubuntu não seja instalado, execute no **PowerShell como administrador**:

```powershell
wsl --install -d Ubuntu
```
 
---

## 2. Instalação do Docker
 
Dentro do terminal Ubuntu, execute os comandos abaixo em sequência:
 
```bash
# Atualiza os pacotes
sudo apt update && sudo apt upgrade -y
 
# Instala dependências
sudo apt install -y ca-certificates curl gnupg lsb-release
 
# Adiciona a chave GPG do Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
 
# Adiciona o repositório do Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
 
# Instala o Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
 
# Adiciona seu usuário ao grupo docker (evita usar sudo toda vez)
sudo usermod -aG docker $USER
```
 
Feche e reabra o terminal, depois teste:
 
```bash
docker ps
```
 
---
 
## 3. Configuração do Docker para iniciar automaticamente
 
Por padrão o Docker não inicia automaticamente no WSL. Para resolver isso, edite o `.bashrc`:
 
```bash
echo 'sudo service docker start' >> ~/.bashrc
```
 
Para não pedir senha toda vez, libere o comando no sudoers:
 
```bash
sudo visudo
```
 
Adicione esta linha no final do arquivo:
 
```
SEU_USUARIO ALL=(ALL) NOPASSWD: /usr/sbin/service docker start
```
 
Substitua `SEU_USUARIO` pelo seu nome de usuário do Ubuntu. Salve com `Ctrl+X`, `Y`, `Enter`.
 
---
 
## 4. Instalação do Portainer
 
O Portainer é a interface visual para gerenciar o Docker.
 
```bash
# Cria o volume para persistir os dados
docker volume create portainer_data
 
# Sobe o container
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts
```
 
Acesse no browser: `http://localhost:9000`
 
No primeiro acesso, o Portainer pedirá um **token de setup**. Para obtê-lo:
 
```bash
docker logs portainer
```
 
Procure a linha com `token:` e cole na tela de setup. Em seguida crie seu usuário administrador.
 
---
 
## 5. Instalação do Mosquitto
 
O Mosquitto é o broker MQTT — intermediário de todas as mensagens IoT.
 
### 5.1 Criar estrutura de pastas
 
```bash
cd ~
mkdir mosquitto
cd mosquitto
mkdir mosquitto_data mosquitto_log config
sudo chown -R 1883:1883 /home/$USER/mosquitto
```
 
### 5.2 Criar o arquivo de configuração
 
```bash
sudo nano mosquitto.conf
```
 
Cole o seguinte conteúdo:
 
```
allow_anonymous true
listener 1883 0.0.0.0
persistence true
persistence_location /mosquitto/data/
```
 
Salve com `Ctrl+X`, `Y`, `Enter`.
 
### 5.3 Subir o container
 
```bash
docker run -d \
  -p 1883:1883 \
  -p 9001:9001 \
  -v $PWD/mosquitto.conf:/mosquitto/config/mosquitto.conf \
  -v $PWD/mosquitto_data:/mosquitto/data \
  -v $PWD/mosquitto_log:/mosquitto/log \
  -v $PWD/config:/mosquitto/config \
  --restart=always \
  --name mosquitto-server \
  eclipse-mosquitto:latest
```
 
### 5.4 Testar o broker
 
Instale o cliente MQTT no Ubuntu:
 
```bash
sudo apt install mosquitto-clients -y
```
 
Abra dois terminais Ubuntu:
 
**Terminal 1 — assinar tópico:**
```bash
mosquitto_sub -h localhost -p 1883 -t "teste"
```
 
**Terminal 2 — publicar mensagem:**
```bash
mosquitto_pub -h localhost -p 1883 -t "teste" -m "hello"
```
 
Se aparecer `hello` no Terminal 1, o broker está funcionando.
 
---
 
## 6. Instalação do Node-RED
 
O Node-RED é o motor de integração — conecta o MQTT ao banco de dados.
 
### 6.1 Criar estrutura de pastas
 
```bash
cd ~
mkdir nodered
cd nodered
mkdir nodered_data
sudo chown -R 1000:1000 /home/$USER/nodered
```
 
### 6.2 Subir o container
 
```bash
docker run -d \
  -p 1880:1880 \
  --name nodered \
  --restart=always \
  -v /home/$USER/nodered/nodered_data:/data \
  nodered/node-red
```
 
Acesse no browser: `http://localhost:1880`
 
---
 
## 7. Instalação do InfluxDB
 
O InfluxDB é o banco de dados de séries temporais — armazena os dados dos sensores com timestamp.
 
```bash
cd ~
mkdir influxdb
 
docker run -d \
  -p 8086:8086 \
  -v $HOME/influxdb:/var/lib/influxdb2 \
  --name influxdb \
  --restart=always \
  influxdb:latest
```
 
Acesse no browser: `http://localhost:8086`
 
No primeiro acesso configure: usuário, senha, organização e bucket.
 
---
 
## 8. Instalação do Grafana
 
O Grafana é a camada de visualização — cria dashboards com os dados do InfluxDB.
 
```bash
docker run -d \
  -p 3000:3000 \
  --name grafana \
  --restart=always \
  grafana/grafana:latest
```
 
Acesse no browser: `http://localhost:3000`
 
Login padrão: `admin` / `admin` — troque a senha no primeiro acesso.
 
---

## 9. Configuração de Rede — ESP32 acessar o broker
 
O ESP32 conecta via WiFi na rede local e precisa alcançar o Mosquitto que roda dentro do WSL.
 
### 9.1 Descobrir o IP do Windows na rede local
 
No CMD do Windows:
 
```cmd
ipconfig
```
 
Anote o **Endereço IPv4** do adaptador Ethernet ou WiFi (ex: `192.168.101.6`). Esse é o IP que o ESP32 usará para se conectar.
 
### 9.2 Descobrir o IP interno do WSL
 
No terminal Ubuntu:
 
```bash
hostname -I
```
 
Anote o primeiro IP (ex: `172.22.46.206`).
 
### 9.3 Liberar a porta no Firewall do Windows
 
No CMD como **administrador**:
 
```cmd
netsh advfirewall firewall add rule name="Mosquitto MQTT" dir=in action=allow protocol=TCP localport=1883
```
 
### 9.4 Criar o proxy de porta do Windows para o WSL
 
No CMD como **administrador** (substitua pelo IP do seu WSL):
 
```cmd
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=1883 connectaddress=172.22.46.206 connectport=1883
```
 
Isso garante que conexões externas (ESP32) cheguem corretamente ao container Mosquitto dentro do WSL.
 
### 9.5 Verificar portas ativas
 
```cmd
netstat -an | findstr 1883
```
 
Deve aparecer `0.0.0.0:1883 LISTENING`.
 
---
