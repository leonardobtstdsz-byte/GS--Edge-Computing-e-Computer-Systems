# 🧠 Assistente Inteligente de Home Office – FutureWork

O FutureWork é um dispositivo inteligente criado como proposta para a matéria acadêmica Edge-Computing-e-Computer-Systems. O dispositivo visa o bem-estar do trabalhador em regime de **Home Office ou Híbrido**, atuando como um verdadeiro **“Guardião da Postura e do Ambiente de Trabalho”**.
Ele monitora automaticamente **postura**, **tempo sentado** e **presença do usuário**, emitindo alertas quando identifica riscos à saúde ou queda de produtividade.

---

## 🎯 Contexto: O Futuro do Trabalho

Com o aumento do trabalho remoto, muitos profissionais passaram a enfrentar problemas ergonômicos e comportamentais, como:

* Fadiga postural
* Longos períodos sentado sem pausas
* Ambiente inadequado
* Redução de produtividade

O FotureWork surge como parte do projeto **FutureWork**, que propõe soluções tecnológicas para tornar o futuro do trabalho mais saudável, sustentável e eficiente.

---

## ⚙️ Como o Dispositivo Funciona

O sistema monitora **três fatores principais**:

### ✔️ 1. Presença do usuário

Detecta automaticamente quando alguém está sentado utilizando um sensor ultrassônico (HC-SR04).

### ✔️ 2. Postura corporal

Um acelerômetro/giroscópio **MPU6050** mede inclinação e identifica postura inadequada (inclinação excessiva para frente ou para trás).

### ✔️ 3. Tempo sentado

O Arduino registra o tempo contínuo sentado.
Após **50 minutos**, o dispositivo emite um alerta solicitando pausa.

### 🔔 Tipos de alertas

* **Postura ruim:** beep curto + LED vermelho
* **Tempo excedido:** beeps longos + LED vermelho
* **Postura ok:** LED verde
* **Stand-by sem usuário:** LED azul

O LCD exibe mensagens como *“POSTURA ERRADA”*, *“PAUSA NECESSÁRIA”*, *“Tempo: XX min”* e *“Aguardando...”*.

---

## 🛠️ Tecnologias e Hardware Utilizados

| Componente                 | Função                                 |
| -------------------------- | -------------------------------------- |
| Arduino Uno                | Processamento central (Edge Computing) |
| MPU6050                    | Monitoramento da postura               |
| HC-SR04                    | Detecção de presença/distância         |
| LCD 16x2 (I2C)             | Feedback visual                        |
| Buzzer / Motor vibratório  | Alertas sonoros                        |
| LED RGB                    | Indicação visual de estado             |
| Jumpers, protoboard, fonte | Estrutura do circuito                  |

---

## 🧩 Estrutura e Lógica do Código

### Principais operações realizadas:

* Leitura da distância para identificar presença
* Cálculo do ângulo corporal via MPU6050
* Cálculo do tempo decorrido sentado
* Emissão de alertas e mensagens no LCD
* Troca de estados (OK, Postura Ruim, Pausa, Stand-by)

---

## 📦 Instalação e Uso

1. Monte o circuito com Arduino Uno, MPU6050, HC-SR04, LCD, LED RGB e Buzzer.
2. Carregue o código na IDE Arduino.
3. Fixe o MPU6050 no encosto da cadeira.
4. Instale o sensor ultrassônico voltado para o usuário.
5. Ligue o dispositivo e use normalmente durante sua rotina.

---

## 🔗 Relação com o Projeto FutureWork

Este dispositivo integra a proposta da plataforma **FutureWork**, funcionando como:

* Ferramenta de ergonomia e saúde
* Monitor inteligente em Edge Computing (privacidade total)
* Complemento físico para a plataforma digital de capacitação e bem-estar

---

## 👨‍💻 Integrantes do Projeto

* **Alexandre Constantino Furtado Júnior** – RM 567188
* **Leonardo Batista de Sousa** – RM 568558
* **Matheus Freitas dos Santos** – RM 567337

---

## 📁 Código Completo

/*
 * PROJETO: Assistente de Home Office (Postura e Tempo)
 * Componentes: Arduino UNO, MPU6050, HC-SR04, LCD I2C, LEDs, Buzzer/Motor.
 */

#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

// --- CONFIGURAÇÕES DE PINOS ---
#define PIN_TRIG 6
#define PIN_ECHO 7
#define PIN_BUZZER 8      // Conecte o Buzzer ou o Motor Vibratório aqui
#define PIN_LED_VERMELHO 9
#define PIN_LED_VERDE 10
#define PIN_LED_AZUL 11

// --- CONFIGURAÇÕES DE PARAMETROS ---
const int DISTANCIA_PRESENCA = 100;   // Distância em cm para considerar que tem alguém sentado
const int LIMITE_TEMPO_MIN = 50;     // Tempo em minutos para pedir pausa (ex: 50 min)
const int TOLERANCIA_POSTURA = 15;   // Graus de tolerância para frente ou trás

// --- OBJETOS ---
LiquidCrystal_I2C lcd(0x27, 16, 2); // Endereço 0x27 é comum, se não for, tente 0x3F
Adafruit_MPU6050 mpu;

// --- VARIÁVEIS GLOBAIS ---
unsigned long tempoInicioTrabalho = 0;
unsigned long ultimoTempo = 0;
bool usuarioPresente = false;
bool alertaAtivo = false;

void setup() {
  Serial.begin(9600);

  // Inicializa Pinos
  pinMode(PIN_TRIG, OUTPUT);
  pinMode(PIN_ECHO, INPUT);
  pinMode(PIN_BUZZER, OUTPUT);
  pinMode(PIN_LED_VERMELHO, OUTPUT);
  pinMode(PIN_LED_VERDE, OUTPUT);
  pinMode(PIN_LED_AZUL, OUTPUT);

  // Inicializa LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Iniciando...");
  delay(2000);

  // Inicializa MPU6050
  if (!mpu.begin()) {
    lcd.clear();
    lcd.print("Erro MPU6050");
    while (1) { 
      digitalWrite(PIN_LED_VERMELHO, HIGH); 
      delay(100); 
      digitalWrite(PIN_LED_VERMELHO, LOW); 
      delay(100);
    }
  }
  
  // Configurações básicas do MPU
  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  mpu.setGyroRange(MPU6050_RANGE_500_DEG);
  mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);

  lcd.clear();
  lcd.print("Sistema Pronto!");
  delay(2000);
  lcd.clear();
}

void loop() {
  // 1. Medir Distância (Verificar se o usuário está na cadeira)
  long distancia = lerUltrassonico();

  if (distancia > 0 && distancia < DISTANCIA_PRESENCA) {
    // USUÁRIO ESTÁ SENTADO
    if (!usuarioPresente) {
      usuarioPresente = true;
      tempoInicioTrabalho = millis(); // Reinicia contagem ou continua? Aqui reinicia ao sentar
    }
    monitorarUsuario();
  } else {
    // USUÁRIO SAIU
    usuarioPresente = false;
    modoStandby();
    // Reseta o tempo se a pessoa sair por mais de 1 minuto (opcional)
    tempoInicioTrabalho = millis(); 
  }
  
  delay(200); // Pequeno delay para estabilidade
}

void monitorarUsuario() {
  // Calcula tempo decorrido em minutos
  unsigned long tempoDecorrido = (millis() - tempoInicioTrabalho) / 60000; 
  
  // --- LER POSTURA (MPU6050) ---
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  // Calcula o ângulo "Pitch" (inclinação frente/trás)
  // Nota: Dependendo de como você colar o sensor, pode precisar ajustar entre X ou Y
  float angulo = atan2(a.acceleration.y, a.acceleration.z) * 180 / PI; 

  // --- LÓGICA DE ALERTA ---
  bool posturaRuim = abs(angulo) > TOLERANCIA_POSTURA;
  bool tempoExcedido = tempoDecorrido >= LIMITE_TEMPO_MIN;

  if (tempoExcedido) {
    tocarAlerta(3); // 3 beeps longos
    digitalWrite(PIN_LED_VERMELHO, HIGH);
    digitalWrite(PIN_LED_VERDE, LOW);
    digitalWrite(PIN_LED_AZUL, LOW);
    
    lcd.setCursor(0, 0);
    lcd.print("PAUSA NECESSARIA");
    lcd.setCursor(0, 1);
    lcd.print("Levante um pouco");
    
  } else if (posturaRuim) {
    tocarAlerta(1); // Beeps curtos
    digitalWrite(PIN_LED_VERMELHO, HIGH);
    digitalWrite(PIN_LED_VERDE, LOW);
    digitalWrite(PIN_LED_AZUL, LOW);

    lcd.setCursor(0, 0);
    lcd.print("POSTURA ERRADA! ");
    lcd.setCursor(0, 1);
    lcd.print("Arrume a coluna ");

  } else {
    // TUDO OK
    digitalWrite(PIN_LED_VERMELHO, LOW);
    digitalWrite(PIN_LED_VERDE, HIGH);
    digitalWrite(PIN_LED_AZUL, LOW);

    lcd.setCursor(0, 0);
    lcd.print("Postura: OK     ");
    lcd.setCursor(0, 1);
    lcd.print("Tempo: ");
    lcd.print(tempoDecorrido);
    lcd.print(" min   ");
  }
}

void modoStandby() {
  digitalWrite(PIN_LED_VERMELHO, LOW);
  digitalWrite(PIN_LED_VERDE, LOW);
  digitalWrite(PIN_LED_AZUL, HIGH); // LED Azul indica Standby
  digitalWrite(PIN_BUZZER, LOW);

  lcd.setCursor(0, 0);
  lcd.print("   Home Office  ");
  lcd.setCursor(0, 1);
  lcd.print(" Aguardando...  ");
}

long lerUltrassonico() {
  digitalWrite(PIN_TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(PIN_TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(PIN_TRIG, LOW);
  
  long duration = pulseIn(PIN_ECHO, HIGH);
  long distance = duration * 0.034 / 2;
  return distance;
}

void tocarAlerta(int tipo) {
  // Tipo 1: Postura (Beep rápido)
  // Tipo 3: Tempo (Beep longo)
  
  // Para não travar o processador, usamos um timer simples sem delay longo
  // Mas para simplificar este exemplo, usaremos tone() rápido
  
  if (tipo == 1) { // Postura
      // Se usar Motor Vibratório: digitalWrite(PIN_BUZZER, HIGH); delay(200); digitalWrite(PIN_BUZZER, LOW);
      // Se usar Buzzer:
      tone(PIN_BUZZER, 1000); 
      delay(100);
      noTone(PIN_BUZZER);
  } 
  else if (tipo == 3) { // Tempo
      // Motor: digitalWrite(PIN_BUZZER, HIGH); delay(500); digitalWrite(PIN_BUZZER, LOW);
      tone(PIN_BUZZER, 1500);
      delay(500);
      noTone(PIN_BUZZER);
  }
}

---
