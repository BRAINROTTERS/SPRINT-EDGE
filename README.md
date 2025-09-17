SPRINT-EDGE

Projeto: Sensor de Velocidade CP4 (Web e Front-end)
Grupo: Brainrotters

👥 Integrantes

RM 12345 — Rafael Moraes Ribeiro dos Santos

RM 562112 — Guilherme Andrade Amaral

RM 562541 — Enrico Bagli Borges

RM 562543 — João Victor Cazarini del Bello

RM 561292 — Matheus Antunes Monreal

📖 Sobre o Projeto

Este projeto foi desenvolvido para o Passa Bola, com os seguintes objetivos:

Criar um sensor de velocidade capaz de calcular a velocidade das atletas;

Implementar um sensor de movimento para identificar a presença de uma pessoa e estimar sua velocidade.

🛠️ Materiais Utilizados

1x ESP32

1x HC-SR04 (sensor ultrassônico)

🌐 Plataformas e Ferramentas

Wokwi → simulação do circuito

Microsoft Azure → programação em nuvem e integração

Postman → testes de API

Cloud (Azure) → armazenamento e processamento de dados

🔗links https://wokwi.com/projects/442274344067011585

Projeto no Wokwi

🚀 Como Rodar o Projeto
1. Simulação no Wokwi

Acesse o link do projeto no Wokwi.

Clique em Run Simulation para rodar o circuito.

O sensor começará a captar os dados de velocidade e presença.

2. Integração com Azure

Configure sua conta no Microsoft Azure.

Conecte o ESP32 ao serviço configurado no Azure IoT Hub.

Os dados coletados serão enviados para a nuvem.

3. Testes com Postman

Abra o Postman.

Realize requisições GET/POST para testar a API que recebe os dados.

Verifique se os valores de velocidade e presença estão sendo retornados corretamente.

Codigo:
#include <WiFi.h>
#include <HTTPClient.h>

// Definições do sensor ultrassônico
#define TRIG_PIN 5      // Pino TRIG do sensor
#define ECHO_PIN 18     // Pino ECHO do sensor

// Limiar de distância para considerar "presença" (em cm)
const int LIMIAR_PRESENCA_CM = 200; // 2 metros

// Constantes para o cálculo
const float VELOCIDADE_DO_SOM = 0.0343; // cm/us (343 m/s)
const unsigned long INTERVALO_MEDICAO_MS = 100; // Mede a cada 100ms

// Variáveis para o Wi-Fi e a API
const char* ssid = "Wokwi-GUEST";     
const char* password = "";
const char* serverUrl = "http://48.211.169.80:1026/v2/entities/urn:ngsi-ld:pir:passabola/attrs/speed";


// Variáveis para o cálculo de velocidade e presença
long duracao;
float distancia_cm;
float distancia_anterior_cm;
unsigned long tempo_anterior_ms;
float velocidade_mps = 0.0;
bool presenca_detectada;

// Objeto para a requisição HTTP
HTTPClient http;

// Função para reconectar ao Wi-Fi com tempo limite
void reconnectWiFi() {
  if (WiFi.status() == WL_CONNECTED) {
    return;
  }
  Serial.println("Tentando conectar ao WiFi...");
  WiFi.begin(ssid, password);
  unsigned long startTime = millis();
  while (WiFi.status() != WL_CONNECTED && (millis() - startTime < 20000)) {
    delay(500);
    Serial.print(".");
  }
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n✅ Conectado ao WiFi!");
  } else {
    Serial.println("\n❌ Falha na conexão com o WiFi. Tentando novamente...");
  }
}

void setup() {
  Serial.begin(115200);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  reconnectWiFi();
  
  tempo_anterior_ms = millis();
  digitalWrite(TRIG_PIN, LOW); delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH); delayMicroseconds(10); digitalWrite(TRIG_PIN, LOW);
  duracao = pulseIn(ECHO_PIN, HIGH, 30000);
  distancia_anterior_cm = duracao * VELOCIDADE_DO_SOM / 2;
}

void loop() {
  if (WiFi.status() != WL_CONNECTED) {
    reconnectWiFi();
    delay(1000);
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    digitalWrite(TRIG_PIN, LOW); delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH); delayMicroseconds(10); digitalWrite(TRIG_PIN, LOW);

    duracao = pulseIn(ECHO_PIN, HIGH, 30000);
    distancia_cm = duracao * VELOCIDADE_DO_SOM / 2;
    
    if (distancia_cm == 0 || distancia_cm > 400) {
      distancia_cm = 400;
      presenca_detectada = false;
      velocidade_mps = 0.0;
    } else {
      presenca_detectada = (distancia_cm <= LIMIAR_PRESENCA_CM);
    }

    unsigned long tempo_atual_ms = millis();
    if (tempo_atual_ms - tempo_anterior_ms >= INTERVALO_MEDICAO_MS) {
      if (distancia_anterior_cm > 0 && distancia_cm > 0) {
        float variacao_distancia_cm = abs(distancia_anterior_cm - distancia_cm);
        float variacao_tempo_seg = (tempo_atual_ms - tempo_anterior_ms) / 1000.0;
        
        velocidade_mps = (variacao_distancia_cm / variacao_tempo_seg) / 100.0;
      }
      distancia_anterior_cm = distancia_cm;
      tempo_anterior_ms = tempo_atual_ms;
    }

    String payload = "{\"presence\":\"" + String(presenca_detectada ? "SIM" : "NAO") + "\"," +
                     "\"speed\":" + String(velocidade_mps, 2) + "}";

    Serial.print("Payload: ");
    Serial.println(payload);

    http.begin(serverUrl);
    http.addHeader("Content-Type", "application/json");
    int httpResponseCode = http.POST(payload);
    Serial.printf("HTTP Code: %d\n", httpResponseCode);
    http.end();
  }
  delay(200);
}
