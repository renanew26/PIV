# PIV
codigo do projeto integrador - Sistema de irrigação inteligente
Para visualizar os dados publicados pelo dispositivo, siga os passos:

1. Acesse http://www.hivemq.com/demos/websocket-client/.
2. Clique em "Connect".
3. Em "Subscriptions", clique em "Add New Topic Subscription".
4. No campo "Topic", digite wokwi-weather e clique em "Subscribe".

Ao alterar a temperatura/umidade no simulador Wokwi (se estiver usando a simulação), as mensagens aparecerão no painel "Messages" do cliente MQTT.

import network
import time
from machine import Pin
import dht
import ujson
from umqtt.simple import MQTTClient

# Parâmetros do Servidor MQTT
MQTT_CLIENT_ID = "micropython-weather-demo"
MQTT_BROKER    = "broker.mqttdashboard.com"
MQTT_USER      = ""
MQTT_PASSWORD  = ""
MQTT_TOPIC     = "wokwi-weather"

# Configuração dos pinos
sensor_ambiente = dht.DHT22(Pin(15))  # Sensor de temperatura e umidade do ambiente no pino 15
sensor_solo = dht.DHT22(Pin(0))       # Sensor de temperatura e umidade do solo no pino 0
bomba_solo = Pin(2, Pin.OUT)          # Relé para irrigação do solo no pino 2
bomba_ambiente = Pin(4, Pin.OUT)      # Segundo relé para controle da umidade do ambiente no pino 4

# Variáveis para armazenar os valores anteriores
prev_temp_ambiente = None
prev_humidity_ambiente = None
prev_temp_solo = None
prev_humidity_solo = None
bomba_solo_status = "Desligado"
bomba_ambiente_status = "Desligado"

# Conexão à rede Wi-Fi
print("Conectando ao WiFi", end="")
sta_if = network.WLAN(network.STA_IF)
sta_if.active(True)
sta_if.connect('Wokwi-GUEST', '')  # SSID e senha da rede Wi-Fi
while not sta_if.isconnected():
  print(".", end="")
  time.sleep(0.1)
print(" Conectado!")

# Conexão ao servidor MQTT
print("Conectando ao servidor MQTT... ", end="")
client = MQTTClient(MQTT_CLIENT_ID, MQTT_BROKER, user=MQTT_USER, password=MQTT_PASSWORD)
client.connect()
print("Conectado!")

while True:
  # Medição das condições do ambiente
  sensor_ambiente.measure()
  temp_ambiente = sensor_ambiente.temperature()
  humidity_ambiente = sensor_ambiente.humidity()
  
  # Medição das condições do solo
  sensor_solo.measure()
  temp_solo = sensor_solo.temperature()
  humidity_solo = sensor_solo.humidity()

  # Verifica se houve mudança nos valores de temperatura ou umidade do ambiente
  if (temp_ambiente != prev_temp_ambiente) or (humidity_ambiente != prev_humidity_ambiente):
    print("Condições do ambiente:")
    print("  - Temperatura: {:.1f} ºC".format(temp_ambiente))
    print("  - Umidade: {:.1f} %".format(humidity_ambiente))
    prev_temp_ambiente = temp_ambiente
    prev_humidity_ambiente = humidity_ambiente
    
    # Envia os dados para o servidor MQTT apenas quando houver mudanças
    message = ujson.dumps({
      "condicoes_ambiente": {
          "temperatura": temp_ambiente,
          "umidade": humidity_ambiente
      },
      "bomba_solo_status": bomba_solo_status,
      "bomba_ambiente_status": bomba_ambiente_status
    })

    print("Reportando para o tópico MQTT {}: {}".format(MQTT_TOPIC, message))
    client.publish(MQTT_TOPIC, message)

    # Verifica a umidade do ambiente e aciona a bomba do ambiente se estiver abaixo de 80%
    if humidity_ambiente < 80:
      print("Umidade do ambiente abaixo de 80%. Ligando a bomba do ambiente.")
      bomba_ambiente.value(1)
      bomba_ambiente_status = "Ligado"
    else:
      print("Umidade do ambiente acima de 80%. Desligando a bomba do ambiente.")
      bomba_ambiente.value(0)
      bomba_ambiente_status = "Desligado"

  # Verifica se houve mudança nos valores de temperatura ou umidade do solo
  if (temp_solo != prev_temp_solo) or (humidity_solo != prev_humidity_solo):
    print("Condições do solo:")
    print("  - Temperatura: {:.1f} ºC".format(temp_solo))
    print("  - Umidade: {:.1f} %".format(humidity_solo))
    prev_temp_solo = temp_solo
    prev_humidity_solo = humidity_solo

    # Verifica se a umidade do solo está abaixo de 65%
    if humidity_solo < 65:
      print("Umidade do solo abaixo de 65%. Ligando a bomba do solo.")
      bomba_solo.value(1)
      bomba_solo_status = "Ligado"
    else:
      print("Umidade do solo acima de 65%. Desligando a bomba do solo.")
      bomba_solo.value(0)
      bomba_solo_status = "Desligado"

    # Envia os dados para o servidor MQTT
    message = ujson.dumps({
      "condicoes_ambiente": {
          "temperatura": temp_ambiente,
          "umidade": humidity_ambiente
      },
      "condicoes_solo": {
          "temperatura": temp_solo,
          "umidade": humidity_solo
      },
      "bomba_solo_status": bomba_solo_status,
      "bomba_ambiente_status": bomba_ambiente_status
    })

    print("Reportando para o tópico MQTT {}: {}".format(MQTT_TOPIC, message))
    client.publish(MQTT_TOPIC, message)

  # Aguarda um segundo antes de fazer a próxima leitura
  time.sleep(1)
