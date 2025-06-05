# 📘 Guia Completo para Implantação de Rede IoT LoRaWAN com ChirpStack

Aluno: Luís Felipe Lewandoski Borsoi

[Repositório](https://github.com/luisfelipe998/lorawan-guia)

[Clique neste link](https://luisfelipe998.github.io/lorawan-guia/) para abrir essa documentação.

## Introdução

Esse documento apresenta um roteiro prático para a implantação de uma rede IoT utilizando o padrão LoRaWAN, com o servidor de rede ChirpStack. O cenário considerado envolve:

* Instalação nativa do ChirpStack em um servidor Ubuntu 22.04
* Uso de um gateway com Raspberry Pi e concentrador SX1302
* Dispositivos finais Heltec WiFi LoRa 32 V2
* Integração de dados via protocolo MQTT

---

## Por que LoRaWAN com ChirpStack?

O LoRaWAN é uma tecnologia de comunicação sem fio ideal para aplicações que exigem longo alcance e baixo consumo de energia. É amplamente usada em agricultura de precisão, cidades inteligentes e monitoramento ambiental. O ChirpStack, por sua vez, é uma solução open-source robusta e flexível para gerenciar redes LoRaWAN sem custos de licenciamento.

### Vantagens

* Código aberto e gratuito
* Compatível com diversas marcas de gateways e dispositivos
* Suporte a múltiplas integrações (MQTT, HTTP, gRPC)
* Excelente alcance de cobertura para áreas amplas

### Desvantagens

* Baixa taxa de transmissão de dados
* Curva de aprendizado inicial acentuada
* Suporte técnico depende da comunidade

---

## Etapa 1 – Configuração do Gateway LoRa (Canais 0–7)

O gateway é o elo entre os dispositivos IoT e o backend da rede. Ele escuta os pacotes LoRa enviados pelos sensores e os retransmite para o servidor ChirpStack via UDP. O uso de canais LoRa configurados corretamente e de um concentrador compatível é essencial para garantir que os dados transmitidos pelos dispositivos cheguem com confiabilidade ao servidor de rede.

O gateway é o elo entre os dispositivos IoT e o backend da rede. Ele escuta os pacotes LoRa enviados pelos sensores e os retransmite para o servidor ChirpStack via UDP.

### Passos:

> Certifique-se de que o serviço `mosquitto` esteja ativo. O broker MQTT escutará por padrão na porta 1883.

> Assume-se a utilização do Ubuntu Server 22.04 com acesso root e conexão com a internet.

Antes de iniciar, certifique-se de utilizar um concentrador SX1301/SX1302/SX1303 devidamente acoplado ao Raspberry Pi, como o RAK2245, RAK2287 ou RAK5146.

1. Baixe a imagem do ChirpStack Gateway OS no site oficial.
2. Grave a imagem em um cartão SD utilizando o Raspberry Pi Imager.
3.  Configure o IP do servidor no arquivo `chirpstack-gateway-os.toml`:
```toml
[gateway]
servers=[{uri="udp://192.168.1.100:1700"}]
```
4. Defina os canais LoRa utilizados no Brasil:
```toml
[concentratord]
multi_sf_channels=[902300000,902500000,902700000,902900000,903100000,903300000,903500000,903700000]
```
5. Inicie o gateway e monitore os logs:
```bash
sudo journalctl -u chirpstack-concentratord -f
```

### Justificativa:

Os canais configurados seguem o padrão brasileiro definido pela ANATEL (faixa ISM 902–928 MHz). A utilização do protocolo Semtech UDP Packet Forwarder garante interoperabilidade com diversos modelos de gateways.

---

## Etapa 2 – Instalação do Servidor ChirpStack (Ubuntu)

Essa etapa estabelece o núcleo lógico da rede LoRaWAN. O servidor ChirpStack processa pacotes, gerencia sessões de dispositivos e permite a configuração de integrações. Esta instalação será feita de forma nativa, o que oferece mais controle e visibilidade do sistema do que ambientes conteinerizados.

O servidor ChirpStack é responsável por gerenciar os dispositivos LoRaWAN, processar os dados recebidos e encaminhá-los para a aplicação cliente.

### Passos:

1. Atualize o sistema e adicione o repositório do ChirpStack:
```bash
sudo apt update
sudo apt install -y curl gnupg lsb-release
curl -fsSL https://artifacts.chirpstack.io/install.sh | sudo bash
```
2. Instale o ChirpStack:
```bash
sudo apt install chirpstack
```
3. Instale e configure o PostgreSQL:
```bash
sudo apt install postgresql
sudo -u postgres psql -c "CREATE ROLE chirpstack WITH LOGIN PASSWORD 'senha';"
sudo -u postgres psql -c "CREATE DATABASE chirpstack WITH OWNER chirpstack;"
```
4. Instale Redis e Mosquitto (broker MQTT):
```bash
sudo apt install redis mosquitto mosquitto-clients
```
5. Inicie e verifique o serviço:
```bash
sudo systemctl enable chirpstack
sudo systemctl start chirpstack
sudo journalctl -u chirpstack -f
```
6. Acesse a interface web:
```
http://<ip-do-servidor>:8080
Usuário: admin | Senha: admin
```

### Justificativa:

* **PostgreSQL** armazena configurações persistentes, sessões, dispositivos e estatísticas. Ele é o banco de dados principal do ChirpStack, garantindo integridade e recuperação de dados, e permite consultas estruturadas sobre o estado da rede, histórico de pacotes e eventos.
* **Redis** atua como intermediador de mensagens entre os serviços internos. Seu papel é servir como uma fila de mensagens e cache de alta performance, permitindo que o Application Server e o Network Server troquem dados com baixa latência.
* **MQTT** é o canal principal de comunicação com as aplicações externas. O Mosquitto é leve e eficiente para isso, suportando múltiplos consumidores em tempo real, como dashboards, scripts de análise e armazenadores de dados. Sua arquitetura pub/sub desacopla os emissores (dispositivos) dos consumidores (clientes). é o canal principal de comunicação com as aplicações externas. O Mosquitto é leve e eficiente para isso.

---

## Etapa 3 – Integração via MQTT

Com o servidor operacional, é necessário configurar a integração com uma aplicação cliente que irá consumir os dados enviados pelos sensores. A integração MQTT é uma das mais utilizadas, pois é leve, eficiente e compatível com a maioria dos sistemas de IoT (como Node-RED, Grafana, Python, InfluxDB etc).

A integração MQTT é utilizada para receber os dados dos sensores LoRaWAN em tempo real, permitindo tratamento por sistemas externos como Node-RED, bancos de dados ou aplicações web.

### Passos:

1. Acesse a interface web do ChirpStack.
2. Crie um **Service Profile** definindo parâmetros de qualidade de serviço (ex: ACKs, delay).
3. Crie um **Device Profile** com região, classe (A/B/C), tipo de ativação (ABP/OTAA).
4. Crie uma **Application**, que irá agrupar os dispositivos e conectar com o MQTT.
5. Registre os dispositivos na aplicação conforme o método de ativação.
6. Teste a integração MQTT com:

```bash
mosquitto_sub -t 'application/1/device/+/event/up' -v
```

### Explicações:

* **Service Profile**: define como a rede se comporta (retransmissões, confirmações, delays, prioridade de downlink). Por exemplo, ele pode especificar se uma aplicação permite downlinks confirmados ou se exige ACKs em uplinks. Isso influencia diretamente no consumo de energia dos dispositivos.
* **Device Profile**: configurações técnicas e de hardware do dispositivo, como classe (A/B/C), taxa de dados, ADR (Adaptive Data Rate), e tipo de ativação (ABP ou OTAA). Ele encapsula todas as capacidades e limitações do dispositivo para o ChirpStack aplicar regras de comunicação corretas.
* **Application**: vínculo lógico entre dispositivos e integrações (MQTT, HTTP, gRPC). É a camada onde se agrupam sensores semelhantes ou que compartilham um mesmo destino de dados. Cada aplicação pode estar conectada a diferentes mecanismos de integração e lógica de negócios.: vínculo lógico entre dispositivos e integrações (MQTT, HTTP...)

---

## Etapa 4 – Dispositivo ABP (Activation By Personalization)

Dispositivos ABP são ideais para ambientes controlados ou testes locais. Eles iniciam imediatamente com chaves estáticas pré-configuradas no código e no ChirpStack, eliminando a necessidade de join. No entanto, por não haver troca dinâmica de chaves, o nível de segurança é menor.

No modo ABP, o dispositivo já inicia com as chaves de sessão pré-definidas, sem necessidade de se registrar dinamicamente na rede.

### Exemplo de código (Arduino/PlatformIO):

```cpp
#include <LoRaWan_APP.h>

void setup() {
  Serial.begin(115200);
  loraWanClass.DevAddr = 0x26011BDA;
  memcpy(loraWanClass.NwkSKey, nwk_key, 16);
  memcpy(loraWanClass.AppSKey, app_key, 16);
  loraWanClass.init();
}

void loop() {
  loraWanClass.send("Ola LoRa", 7);
  delay(60000);
}
```

### Cadastro no ChirpStack:

* Configure DevAddr, NwkSKey e AppSKey manualmente. O **DevAddr** é o endereço lógico do dispositivo na rede. O **NwkSKey** (Network Session Key) é usado para autenticação e verificação da integridade das mensagens LoRaWAN no nível de rede, enquanto o **AppSKey** (Application Session Key) protege a confidencialidade do payload da aplicação. Esses valores devem ser exatamente os mesmos entre o código do dispositivo e a configuração no ChirpStack.
* Associe o Device Profile com ativação ABP, garantindo que o dispositivo use um modo sem join, com sessões pré-estabelecidas..
* Associe o Device Profile com ativação ABP.

### Prós e Contras

* ✅ Simplicidade: o dispositivo pode começar a operar imediatamente, sem depender de processos adicionais de autenticação ou autorização.
* ✅ Ideal para ambientes de teste, onde a segurança não é uma prioridade e o controle é local.
* ✅ Permite operação offline total, já que não requer troca com o servidor para iniciar.
* ❌ Menor segurança: as chaves são fixas e reutilizadas, o que as torna vulneráveis caso sejam interceptadas.
* ❌ Difícil manutenção em larga escala: qualquer mudança de chave exige reprogramação manual do dispositivo.
* ❌ Não suporta rotação de chaves ou controle dinâmico por parte do servidor.

---

## Etapa 5 – Dispositivo OTAA (Over-The-Air Activation)

OTAA é o método recomendado para ambientes em produção. Nele, o dispositivo realiza um processo de 'join' na rede e, se autorizado, recebe chaves de sessão válidas. Isso permite mais segurança e controle, com suporte à renovação dinâmica de chaves.

### Exemplo de código:

```cpp
#include <LoRaWan_APP.h>

void setup() {
  Serial.begin(115200);
  loraWanClass.joinOTAA(dev_eui, app_eui, app_key);
}

void loop() {
  if (loraWanClass.isJoined()) {
    loraWanClass.send("Ola LoRa OTAA", 15);
  }
  delay(60000);
}
```

### Cadastro no ChirpStack:

* Informe **DevEUI** (identificador único do dispositivo), **AppEUI** (identificador da aplicação) e **AppKey** (chave de autenticação para o processo de join). Estes dados são utilizados pelo servidor para verificar a autenticidade do dispositivo e gerar dinamicamente as chaves de sessão (NwkSKey e AppSKey).
* Ative a opção "Device supports OTAA" no Device Profile, garantindo que o ChirpStack trate esse dispositivo como dinâmico, permitindo o join remoto e a renovação automática de sessão em caso de reconexão ou expiração.

### Prós e Contras

* ✅ Alta segurança: as chaves de sessão (NwkSKey, AppSKey) são geradas dinamicamente durante o join, dificultando interceptações.
* ✅ Suporte à renovação automática de chaves, aumentando a resiliência e o controle da rede.
* ✅ Flexibilidade: ideal para ambientes distribuídos e produção em larga escala, com dispositivos que podem se mover entre redes.
* ❌ Requer que o dispositivo consiga realizar o join com sucesso, o que pode ser afetado por interferência ou distância do gateway.
* ❌ Processo de inicialização levemente mais demorado e dependente de conectividade para funcionar corretamente.
* ❌ Pode consumir mais energia se múltiplas tentativas de join forem necessárias.

## Conclusão

Este guia reúne os principais passos para implantar uma rede LoRaWAN funcional com ChirpStack, utilizando integração via MQTT e dispositivos Heltec. Essa abordagem oferece baixo custo, escalabilidade e alta flexibilidade para projetos de IoT em diversas áreas.

Para ambientes mais robustos, considere os seguintes pontos:

* Utilizar Docker ou Kubernetes para escalar os serviços
* Configurar autenticação TLS para o broker MQTT
* Implementar VPNs entre gateways e servidor central

---

## Referências

* [ChirpStack Documentation](https://www.chirpstack.io/)
* [Heltec ESP32 Docs](https://docs.heltec.org/)
* [Anatel – Faixa ISM](https://www.anatel.gov.br/)
