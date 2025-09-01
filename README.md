# Relogio-Wifi-AP
# Relogio digital com acesso as configurações por server web via Lan ou AP.

# Não sei usar direito aqui.

# codigo>>

/*
  relogio_test.ino
  Relógio ESP32-CYD 2.4" (TFT_eSPI)
  - WebServer com formulário (WiFi, AP, NTP, hora/data manual, cor, formato)
  - AP só fica ligado se NÃO conectar ao WiFi
  - Atualização sem piscar
  - Salva configurações em Preferences
  - Suporte a ajuste manual de hora/data/semana
*/

#include <WiFi.h>
#include <WebServer.h>
#include <Preferences.h>
#include <TFT_eSPI.h>
#include <SPI.h>
#include <time.h>

// ---------- Objetos ----------
TFT_eSPI tft = TFT_eSPI();
Preferences prefs;
WebServer server(80);

// ---------- Configs ----------
String ssid = "";
String senha = "";
String ssidAP = "Relogio_AP";
String senhaAP = "12345678";
String ntpServer = "a.st1.ntp.br";
String formato = "24h";
String corSelecionada = "Branco";
bool usarNTP = true;

// ---------- Configuração Manual ----------
int manualDia = 1, manualMes = 1, manualAno = 2025;
int manualHora = 0, manualMin = 0, manualSeg = 0;
int manualSemana = 0; // 0=Domingo ... 6=Sábado

// ---------- Posicionamento ----------
int xIP = 10, yIP = 8;
int xData = 240, yData = 8;
int xSemana = 240, ySemana = 30;
int xHora = 160, yHora = 110;
int xClima = 160, yClima = 210;

// ---------- Fontes ----------
int fontIP = 2;
int fontData = 2;
int fontSemana = 2;
int fontHora = 7;
int fontClima = 2;

// ---------- Estado ----------
bool conectadoWiFi = false;
uint16_t corTexto = TFT_WHITE;
unsigned long ultimoUpdate = 0;

// ---------- Cache ----------
String cacheHora = "";
String cacheData = "";
String cacheSemana = "";
String cacheIP = "";
String cacheClima = "";

// ---------- Timezone ----------
const long gmtOffset_sec = -3 * 3600;
const int daylightOffset_sec = 0;

uint16_t getColorFromName(const String &nome) {
  if (nome == "Vermelho") return TFT_RED;
  if (nome == "Verde") return TFT_GREEN;
  if (nome == "Azul") return TFT_BLUE;
  if (nome == "Amarelo") return TFT_YELLOW;
  return TFT_WHITE;
}

// ---------- Preferences ----------
void savePreferences() {
  prefs.begin("config", false);
  prefs.putString("ssid", ssid);
  prefs.putString("senha", senha);
  prefs.putString("ssidap", ssidAP);
  prefs.putString("senhaap", senhaAP);
  prefs.putString("ntp", ntpServer);
  prefs.putString("formato", formato);
  prefs.putString("cor", corSelecionada);
  prefs.putBool("usntp", usarNTP);

  prefs.putInt("m_dia", manualDia);
  prefs.putInt("m_mes", manualMes);
  prefs.putInt("m_ano", manualAno);
  prefs.putInt("m_hora", manualHora);
  prefs.putInt("m_min", manualMin);
  prefs.putInt("m_seg", manualSeg);
  prefs.putInt("m_semana", manualSemana);
  prefs.end();
}

// ---------- Setar Hora Manual ----------
void setManualTime() {
  struct tm t = {};
  t.tm_year = manualAno - 1900;
  t.tm_mon = manualMes - 1;
  t.tm_mday = manualDia;
  t.tm_hour = manualHora;
  t.tm_min = manualMin;
  t.tm_sec = manualSeg;
  t.tm_wday = manualSemana;
  time_t tt = mktime(&t);
  struct timeval now = { .tv_sec = tt };
  settimeofday(&now, NULL);
}

// ---------- Web Pages ----------
String pageForm() {
  String semanaOpts = "";
  const char *dias[] = {"Domingo","Segunda","Terça","Quarta","Quinta","Sexta","Sábado"};
  for (int i=0;i<7;i++) {
    semanaOpts += "<option value='"+String(i)+"' "+(manualSemana==i?"selected":"")+">"+dias[i]+"</option>";
  }

  String html = "<!doctype html><html lang='pt-BR'><head><meta charset='utf-8'>"
                "<meta name='viewport' content='width=device-width, initial-scale=1'>"
                "<title>Servidor do Relógio</title></head><body style='font-family:Arial;padding:12px'>"
                "<h2>Servidor do Relógio</h2>"
                "<form method='POST' action='/salvar'>"

                "<fieldset><legend><b>WiFi (STA)</b></legend>"
                "SSID:<br><input name='ssid' value='" + ssid + "'><br>"
                "Senha:<br><input type='password' name='senha' value='" + senha + "'><br></fieldset><br>"

                "<fieldset><legend><b>Access Point (AP)</b></legend>"
                "SSID AP:<br><input name='ssidap' value='" + ssidAP + "'><br>"
                "Senha AP:<br><input type='password' name='senhaap' value='" + senhaAP + "'><br></fieldset><br>"

                "<fieldset><legend><b>NTP / Hora</b></legend>"
                "Servidor NTP:<br><input name='ntp' value='" + ntpServer + "'><br>"
                "Usar NTP: <input type='checkbox' name='usntp' value='1' " + String(usarNTP ? "checked" : "") + "><br><br>"
                "Formato hora: <select name='formato'>"
                "<option value='24h' " + String(formato == "24h" ? "selected" : "") + ">24h</option>"
                "<option value='12h' " + String(formato == "12h" ? "selected" : "") + ">12h</option>"
                "</select><br></fieldset><br>"

                "<fieldset><legend><b>Configuração Manual</b></legend>"
                "Data (dd/mm/aaaa): <input name='m_dia' size=2 value='"+String(manualDia)+"'> / "
                "<input name='m_mes' size=2 value='"+String(manualMes)+"'> / "
                "<input name='m_ano' size=4 value='"+String(manualAno)+"'><br><br>"

                "Hora (hh:mm:ss): <input name='m_hora' size=2 value='"+String(manualHora)+"'> : "
                "<input name='m_min' size=2 value='"+String(manualMin)+"'> : "
                "<input name='m_seg' size=2 value='"+String(manualSeg)+"'><br><br>"

                "Dia da semana: <select name='m_semana'>" + semanaOpts + "</select><br></fieldset><br>"

                "<fieldset><legend><b>Aparência</b></legend>"
                "Cor da fonte: <select name='cor'>"
                "<option value='Branco' " + String(corSelecionada == "Branco" ? "selected" : "") + ">Branco</option>"
                "<option value='Vermelho' " + String(corSelecionada == "Vermelho" ? "selected" : "") + ">Vermelho</option>"
                "<option value='Verde' " + String(corSelecionada == "Verde" ? "selected" : "") + ">Verde</option>"
                "<option value='Azul' " + String(corSelecionada == "Azul" ? "selected" : "") + ">Azul</option>"
                "<option value='Amarelo' " + String(corSelecionada == "Amarelo" ? "selected" : "") + ">Amarelo</option>"
                "</select><br></fieldset><br>"

                "<input type='submit' value='Salvar'>"
                "</form></body></html>";
  return html;
}

void handleRoot() {
  server.send(200, "text/html; charset=UTF-8", pageForm());
}

void handleSalvar() {
  ssid = server.arg("ssid");
  senha = server.arg("senha");
  ssidAP = server.arg("ssidap");
  senhaAP = server.arg("senhaap");
  ntpServer = server.arg("ntp");
  formato = server.arg("formato");
  corSelecionada = server.arg("cor");
  usarNTP = server.hasArg("usntp");

  manualDia = server.arg("m_dia").toInt();
  manualMes = server.arg("m_mes").toInt();
  manualAno = server.arg("m_ano").toInt();
  manualHora = server.arg("m_hora").toInt();
  manualMin = server.arg("m_min").toInt();
  manualSeg = server.arg("m_seg").toInt();
  manualSemana = server.arg("m_semana").toInt();

  savePreferences();
  if (!usarNTP) setManualTime();

  server.send(200, "text/html; charset=UTF-8", "<h3>Configuração salva!</h3><p>Reiniciando...</p>");
  delay(800);
  ESP.restart();
}

// ---------- WiFi / AP ----------
bool tryConnectWiFiWithRetries(int attempts = 10, int delayMs = 5000) {
  if (ssid.length() == 0) return false;
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid.c_str(), senha.c_str());
  for (int i = 1; i <= attempts; ++i) {
    if (WiFi.waitForConnectResult() == WL_CONNECTED) {
      conectadoWiFi = true;
      return true;
    }
    delay(delayMs);
  }
  conectadoWiFi = false;
  return false;
}

void startAP() {
  WiFi.mode(WIFI_AP);
  WiFi.softAP(ssidAP.c_str(), senhaAP.c_str());
}

void stopAP() {
  WiFi.softAPdisconnect(true);
  WiFi.mode(WIFI_STA);
}

// ---------- Tela ----------
void drawInitialLayout() {
  tft.fillScreen(TFT_BLACK);
}

void updateIPOnScreen() {
  String ipstr;
  if (conectadoWiFi) ipstr = "IP: " + WiFi.localIP().toString();
  else if (WiFi.getMode() & WIFI_MODE_AP) ipstr = "AP: 192.168.4.1";
  else ipstr = "Conectando...";

  if (ipstr != cacheIP) {
    tft.fillRect(xIP, yIP, 200, 18, TFT_BLACK);
    tft.setTextColor(corTexto, TFT_BLACK);
    tft.setTextFont(fontIP);
    tft.drawString(ipstr, xIP, yIP);
    cacheIP = ipstr;
  }
}

void updateDateTimeOnScreen() {
  struct tm timeinfo;
  char buf[32];
  if (getLocalTime(&timeinfo)) {
    strftime(buf, sizeof(buf), "%d/%m/%Y", &timeinfo);
    String datas = String(buf);
    if (datas != cacheData) {
      tft.fillRect(xData-80, yData-2, 160, 18, TFT_BLACK);
      tft.setTextColor(corTexto, TFT_BLACK);
      tft.setTextFont(fontData);
      tft.drawCentreString(datas, xData, yData, fontData);
      cacheData = datas;
    }

    strftime(buf, sizeof(buf), "%A", &timeinfo);
    String semana = String(buf);
    if (semana != cacheSemana) {
      tft.fillRect(xSemana-80, ySemana-2, 160, 18, TFT_BLACK);
      tft.setTextColor(corTexto, TFT_BLACK);
      tft.setTextFont(fontSemana);
      tft.drawCentreString(semana, xSemana, ySemana, fontSemana);
      cacheSemana = semana;
    }

    if (formato=="24h") strftime(buf,sizeof(buf),"%H:%M:%S",&timeinfo);
    else strftime(buf,sizeof(buf),"%I:%M:%S %p",&timeinfo);
    String hora=String(buf);
    if (hora!=cacheHora) {
      tft.fillRect(xHora-100, yHora-20, 200, 60, TFT_BLACK);
      tft.setTextColor(corTexto, TFT_BLACK);
      tft.setTextFont(fontHora);
      tft.drawCentreString(hora, xHora, yHora, fontHora);
      cacheHora=hora;
    }
  }
}

// ---------- Setup ----------
void setup() {
  Serial.begin(115200);
  tft.init();
  tft.setRotation(1);

  prefs.begin("config", true);
  ssid = prefs.getString("ssid", "");
  senha = prefs.getString("senha", "");
  ssidAP = prefs.getString("ssidap", ssidAP);
  senhaAP = prefs.getString("senhaap", senhaAP);
  ntpServer = prefs.getString("ntp", ntpServer);
  formato = prefs.getString("formato", formato);
  corSelecionada = prefs.getString("cor", corSelecionada);
  usarNTP = prefs.getBool("usntp", usarNTP);
  manualDia = prefs.getInt("m_dia", manualDia);
  manualMes = prefs.getInt("m_mes", manualMes);
  manualAno = prefs.getInt("m_ano", manualAno);
  manualHora = prefs.getInt("m_hora", manualHora);
  manualMin = prefs.getInt("m_min", manualMin);
  manualSeg = prefs.getInt("m_seg", manualSeg);
  manualSemana = prefs.getInt("m_semana", manualSemana);
  prefs.end();

  corTexto = getColorFromName(corSelecionada);
  drawInitialLayout();

  if (ssid.length()>0 && tryConnectWiFiWithRetries()) {
    stopAP();
    if (usarNTP) configTime(gmtOffset_sec, daylightOffset_sec, ntpServer.c_str());
    else setManualTime();
  } else {
    startAP();
    if (!usarNTP) setManualTime();
  }

  server.on("/", HTTP_GET, handleRoot);
  server.on("/salvar", HTTP_POST, handleSalvar);
  server.begin();
}

// ---------- Loop ----------
void loop() {
  server.handleClient();
  if (millis()-ultimoUpdate>900) {
    updateIPOnScreen();
    updateDateTimeOnScreen();
    ultimoUpdate=millis();
  }
}
