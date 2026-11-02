# Bot-Rescue
BOT RESCUE é um TCC escolar do curso técnico em Eletrônica da FAETEC ETEJLN, desenvolvido pelas alunas Giovanna Maia e Ana Carolina. O projeto consiste na criação de um robô móvel para apoio à inspeção térmica em áreas de risco, utilizando ESP32, sensor MLX90614 e controle por aplicativo via Wi-Fi.
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <Adafruit_MLX90614.h>

// =====================================================
// MOTORES - L298N
// =====================================================

const int in1 = 26;
const int in2 = 25;
const int in3 = 33;
const int in4 = 32;

// =====================================================
// MLX90614
// =====================================================

const int SDA_PIN = 21;
const int SCL_PIN = 22;

Adafruit_MLX90614 mlx = Adafruit_MLX90614();

// Faixa inicial de temperatura
const float TEMP_MIN = 29.0;
const float TEMP_MAX = 42.0;

// Quantidade de leituras consecutivas para confirmar
const int LEITURAS_CONFIRMAR = 5;

int leiturasValidas = 0;

// =====================================================
// LED RGB
// =====================================================

const int LED_RED = 12;
const int LED_GREEN = 13;
const int LED_BLUE = 14;

// =====================================================
// BUZZER
// =====================================================

const int BUZZER = 27;

// =====================================================
// WI-FI
// =====================================================

const char* ssid = "ESP32_CAR";
const char* password = "12345678";

WebServer server(80);

// =====================================================
// CONTROLE DE STATUS
// =====================================================

enum Estado {
  PROCURANDO,
  NAO_ENCONTRADO,
  ENCONTRADO
};

Estado estadoAtual = PROCURANDO;

// Controle do tempo sem encontrar
unsigned long inicioBusca = 0;

// Tempo que o vermelho fica ligado
unsigned long inicioVermelho = 0;

const unsigned long TEMPO_SEM_DETECCAO = 60000; // 1 minuto
const unsigned long TEMPO_VERMELHO = 5000;      // 5 segundos

// =====================================================
// FUNÇÕES DO LED
// =====================================================

void ledAzul() {
  digitalWrite(LED_RED, LOW);
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(LED_BLUE, HIGH);
}

void ledVerde() {
  digitalWrite(LED_RED, LOW);
  digitalWrite(LED_GREEN, HIGH);
  digitalWrite(LED_BLUE, LOW);
}

void ledVermelho() {
  digitalWrite(LED_RED, HIGH);
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(LED_BLUE, LOW);
}

void ledApagado() {
  digitalWrite(LED_RED, LOW);
  digitalWrite(LED_GREEN, LOW);
  digitalWrite(LED_BLUE, LOW);
}

// =====================================================
// MOTORES
// =====================================================

void forward() {
  digitalWrite(in1, LOW);
  digitalWrite(in2, HIGH);

  digitalWrite(in3, LOW);
  digitalWrite(in4, HIGH);

  server.send(200, "text/plain", "Forward");
}

void backward() {
  digitalWrite(in1, HIGH);
  digitalWrite(in2, LOW);

  digitalWrite(in3, HIGH);
  digitalWrite(in4, LOW);

  server.send(200, "text/plain", "Backward");
}

void left() {
  digitalWrite(in1, HIGH);
  digitalWrite(in2, LOW);

  digitalWrite(in3, LOW);
  digitalWrite(in4, HIGH);

  server.send(200, "text/plain", "Left");
}

void right() {
  digitalWrite(in1, LOW);
  digitalWrite(in2, HIGH);

  digitalWrite(in3, HIGH);
  digitalWrite(in4, LOW);

  server.send(200, "text/plain", "Right");
}

void stopCar() {
  digitalWrite(in1, LOW);
  digitalWrite(in2, LOW);

  digitalWrite(in3, LOW);
  digitalWrite(in4, LOW);

  server.send(200, "text/plain", "Stop");
}

// =====================================================
// ATUALIZA DETECÇÃO
// =====================================================

void atualizarSensor() {

  float temperatura = mlx.readObjectTempC();

  bool temperaturaValida =
    temperatura >= TEMP_MIN &&
    temperatura <= TEMP_MAX;

  // ---------------------------------------------------
  // Encontrou temperatura compatível
  // ---------------------------------------------------

  if (temperaturaValida) {

    leiturasValidas++;

    // Só confirma depois de várias leituras
    if (leiturasValidas >= LEITURAS_CONFIRMAR) {

      estadoAtual = ENCONTRADO;

      ledVerde();

      digitalWrite(BUZZER, HIGH);

      // Reinicia o contador de busca
      inicioBusca = millis();
    }
  }

  // ---------------------------------------------------
  // Temperatura fora da faixa
  // ---------------------------------------------------

  else {

    leiturasValidas = 0;

    digitalWrite(BUZZER, LOW);

    // Se estava encontrado e deixou de detectar,
    // volta para procurar
    if (estadoAtual == ENCONTRADO) {

      estadoAtual = PROCURANDO;

      ledAzul();

      inicioBusca = millis();
    }
  }

  // ---------------------------------------------------
  // Depois de 1 minuto sem encontrar
  // ---------------------------------------------------

  if (estadoAtual == PROCURANDO) {

    if (millis() - inicioBusca >= TEMPO_SEM_DETECCAO) {

      estadoAtual = NAO_ENCONTRADO;

      ledVermelho();

      inicioVermelho = millis();
    }
  }

  // ---------------------------------------------------
  // Vermelho fica 5 segundos
  // ---------------------------------------------------

  if (estadoAtual == NAO_ENCONTRADO) {

    if (millis() - inicioVermelho >= TEMPO_VERMELHO) {

      estadoAtual = PROCURANDO;

      ledAzul();

      inicioBusca = millis();
    }
  }
}

// =====================================================
// ENVIA INFORMAÇÕES PARA O SITE
// =====================================================

void enviarStatus() {

  float temperatura = mlx.readObjectTempC();

  String statusTexto;

  if (estadoAtual == PROCURANDO) {
    statusTexto = "Procurando...";
  }
  else if (estadoAtual == NAO_ENCONTRADO) {
    statusTexto = "Não encontrado";
  }
  else {
    statusTexto = "Encontrado!";
  }

  // Verifica se existe algum celular conectado
  bool wifiConectado = WiFi.softAPgetStationNum() > 0;

  String wifiTexto;

  if (wifiConectado) {
    wifiTexto = "Conectado";
  } else {
    wifiTexto = "Desconectado";
  }

  String json = "{";

  json += "\"temperatura\":";
  json += String(temperatura, 1);

  json += ",\"status\":\"";
  json += statusTexto;
  json += "\"";

  json += ",\"wifi\":\"";
  json += wifiTexto;
  json += "\"";

  json += ",\"estado\":";
  json += String((int)estadoAtual);

  json += "}";

  server.send(200, "application/json", json);
}

// =====================================================
// INTERFACE
// =====================================================

void handleRoot() {

  String html = R"rawliteral(

<!DOCTYPE html>

<html lang="pt-BR">

<head>

<meta charset="UTF-8">

<meta name="viewport"
content="width=device-width, initial-scale=1.0">

<title>BOT RESCUE</title>

<style>

* {
  box-sizing: border-box;
  -webkit-tap-highlight-color: transparent;
}

body {

  margin: 0;

  background: #000;

  color: white;

  font-family: Arial, Helvetica, sans-serif;

  text-align: center;

}

/* =================================================
   TOPO
   ================================================= */

.topo {

  width: 100%;

  height: 25px;

  background: #0047AB;

}

/* =================================================
   LOGO
   ================================================= */

.logo {

  width: 100%;

  display: flex;

  justify-content: center;

  margin-top: 0;

}

.logo img {

  max-width: 418px;

  width: 46%;

  height: auto;

}

/* =================================================
   INFORMAÇÕES
   ================================================= */

.informacoes {

  width: 90%;

  max-width: 1000px;

  margin: 45px auto 20px auto;

}

.info {

  background: #373737;

  border-radius: 50px;

  height: 75px;

  margin-bottom: 35px;

  display: flex;

  align-items: center;

  text-align: left;

  padding: 0 25px;

  font-size: 42px;

}

.info .valor {

  margin-left: 30px;

}

/* =================================================
   PAINEL DOS CONTROLES
   ================================================= */

.controles {

  width: 94%;

  max-width: 1100px;

  margin: 25px auto;

  background: #373737;

  border-radius: 45px;

  padding: 35px 25px;

  display: grid;

  grid-template-columns: 1fr 1.1fr 1fr;

  grid-template-rows: 180px 300px 180px;

  gap: 25px;

}

/* =================================================
   BOTÕES
   ================================================= */

.botao {

  background: #ff711c;

  border: none;

  border-radius: 70px;

  color: #000;

  font-size: 36px;

  cursor: pointer;

  touch-action: manipulation;

  transition: transform 0.1s, opacity 0.1s;

}

.botao:active {

  transform: scale(0.95);

  opacity: 0.75;

}

/* Frente */

#forward {

  grid-column: 2;

  grid-row: 1;

}

/* Esquerda */

#left {

  grid-column: 1;

  grid-row: 2;

}

/* Direita */

#right {

  grid-column: 3;

  grid-row: 2;

}

/* Trás */

#backward {

  grid-column: 2;

  grid-row: 3;

}

/* Parar */

.parar {

  grid-column: 2;

  grid-row: 2;

  width: 230px;

  height: 230px;

  border-radius: 50%;

  background: #0057c8;

  color: #ff711c;

  font-size: 38px;

  justify-self: center;

  align-self: center;

}

/* =================================================
   FOTOS
   ================================================= */

.fotos {

  display: flex;

  justify-content: center;

  align-items: center;

  gap: 25px;

  margin: 25px auto;

  width: 94%;

  overflow: hidden;

}

.fotos img {

  width: 22%;

  height: 150px;

  object-fit: cover;

}

/* =================================================
   CELULAR
   ================================================= */

@media (max-width: 700px) {

  .logo img {

    width: 65%;

  }

  .informacoes {

    margin-top: 30px;

  }

  .info {

    height: 60px;

    font-size: 25px;

    margin-bottom: 20px;

    padding: 0 20px;

  }

  .info .valor {

    margin-left: 15px;

  }

  .controles {

    grid-template-columns: 1fr 1.1fr 1fr;

    grid-template-rows: 130px 230px 130px;

    gap: 12px;

    padding: 20px 12px;

    border-radius: 30px;

  }

  .botao {

    font-size: 22px;

    border-radius: 40px;

  }

  .parar {

    width: 150px;

    height: 150px;

    font-size: 25px;

  }

  .fotos {

    gap: 8px;

  }

  .fotos img {

    height: 80px;

  }

}

</style>

</head>

<body>

<div class="topo"></div>

<!-- =================================================
     LOGO
     ================================================= -->

<div class="logo">

  <!--
  A logo abaixo usa a imagem do projeto.
  Depois podemos colocar a imagem diretamente no ESP32
  ou substituir por HTML/CSS.
  -->

  <div style="
    background:#151515;
    width:418px;
    max-width:46%;
    padding:25px;
    font-size:55px;
    font-weight:bold;
  ">

    <span style="color:white;">BOT</span>
    <br>
    <span style="color:#ff6600;">RESCUE</span>

  </div>

</div>

<!-- =================================================
     INFORMAÇÕES
     ================================================= -->

<div class="informacoes">

  <div class="info">

    Temperatura:

    <span class="valor" id="temperatura">
      -- °C
    </span>

  </div>

  <div class="info">

    Status:

    <span class="valor" id="status">
      Procurando...
    </span>

  </div>

  <div class="info">

    Wi-fi:

    <span class="valor" id="wifi">
      Desconectado
    </span>

  </div>

</div>

<!-- =================================================
     CONTROLES
     ================================================= -->

<div class="controles">

  <button class="botao"
          id="forward">
    Frente
  </button>

  <button class="botao"
          id="left">
    Esquerda
  </button>

  <button class="parar"
          id="stop">
    Parar
  </button>

  <button class="botao"
          id="right">
    Direita
  </button>

  <button class="botao"
          id="backward">
    Trás
  </button>

</div>

<!-- =================================================
     FOTOS
     ================================================= -->

<div class="fotos">

  <img src="https://i.imgur.com/placeholder.jpg">

  <img src="https://i.imgur.com/placeholder.jpg">

  <img src="https://i.imgur.com/placeholder.jpg">

  <img src="https://i.imgur.com/placeholder.jpg">

</div>

<script>

/* =================================================
   CONTROLE DOS MOTORES
   ================================================= */

function enviarComando(comando) {

  fetch("/" + comando);

}

/*
   Mantemos a correção que fizemos anteriormente:

   Frente  -> R
   Trás    -> L
   Esquerda -> B
   Direita -> F
*/

function configurarBotao(id, comando) {

  const botao = document.getElementById(id);

  botao.addEventListener("mousedown", function() {
    enviarComando(comando);
  });

  botao.addEventListener("mouseup", function() {
    enviarComando("S");
  });

  botao.addEventListener("mouseleave", function() {
    enviarComando("S");
  });

  botao.addEventListener("touchstart", function(e) {

    e.preventDefault();

    enviarComando(comando);

  });

  botao.addEventListener("touchend", function(e) {

    e.preventDefault();

    enviarComando("S");

  });

}

/* Botões */

configurarBotao("forward", "R");

configurarBotao("backward", "L");

configurarBotao("left", "B");

configurarBotao("right", "F");


/* Parar */

document.getElementById("stop").addEventListener(
  "click",
  function() {

    enviarComando("S");

  }
);


/* =================================================
   ATUALIZA INFORMAÇÕES
   ================================================= */

function atualizarInformacoes() {

  fetch("/status")

    .then(response => response.json())

    .then(data => {

      /* Temperatura */

      document.getElementById("temperatura").innerHTML =
        data.temperatura.toFixed(1) + " °C";


      /* Status */

      document.getElementById("status").innerHTML =
        data.status;


      /* Wi-Fi */

      document.getElementById("wifi").innerHTML =
        data.wifi;

    })

    .catch(error => {

      document.getElementById("wifi").innerHTML =
        "Desconectado";

    });

}


/*
   Atualiza a cada 500 ms
   = 2 vezes por segundo
*/

setInterval(atualizarInformacoes, 500);


/* Atualiza imediatamente */

atualizarInformacoes();

</script>

</body>

</html>

)rawliteral";

  server.send(200, "text/html", html);
}

// =====================================================
// SETUP
// =====================================================

void setup() {

  Serial.begin(115200);

  // ---------------------------------------------------
  // Motores
  // ---------------------------------------------------

  pinMode(in1, OUTPUT);
  pinMode(in2, OUTPUT);
  pinMode(in3, OUTPUT);
  pinMode(in4, OUTPUT);

  stopCar();

  // ---------------------------------------------------
  // LED
  // ---------------------------------------------------

  pinMode(LED_RED, OUTPUT);
  pinMode(LED_GREEN, OUTPUT);
  pinMode(LED_BLUE, OUTPUT);

  ledAzul();

  // ---------------------------------------------------
  // Buzzer
  // ---------------------------------------------------

  pinMode(BUZZER, OUTPUT);

  digitalWrite(BUZZER, LOW);

  // ---------------------------------------------------
  // MLX90614
  // ---------------------------------------------------

  Wire.begin(SDA_PIN, SCL_PIN);

  if (!mlx.begin()) {

    Serial.println("ERRO: MLX90614 nao encontrado!");

  } else {

    Serial.println("MLX90614 encontrado!");

  }

  // ---------------------------------------------------
  // Wi-Fi
  // ---------------------------------------------------

  WiFi.softAP(ssid, password);

  Serial.println();
  Serial.println("================================");
  Serial.println("BOT RESCUE");
  Serial.println("Wi-Fi iniciado!");
  Serial.print("IP: ");
  Serial.println(WiFi.softAPIP());
  Serial.println("================================");

  // ---------------------------------------------------
  // Rotas
  // ---------------------------------------------------

  server.on("/", handleRoot);

  server.on("/F", forward);
  server.on("/B", backward);
  server.on("/L", left);
  server.on("/R", right);
  server.on("/S", stopCar);

  // Rota das informações
  server.on("/status", enviarStatus);

  server.begin();

  // Começa contagem dos 60 segundos
  inicioBusca = millis();
}

// =====================================================
// LOOP
// =====================================================

void loop() {

  server.handleClient();

  atualizarSensor();

}
