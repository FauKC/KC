#include <PubSubClient.h>
#include <ESP8266Firebase.h>
#include <NTPClient.h>
#include <ESP8266WiFi.h>
#include <WiFiUdp.h>
#include <Callmebot_ESP8266.h>

WiFiUDP ntpUDP;
NTPClient timeClient(ntpUDP, "id.pool.ntp.org", 25200);

const char* ssid = "Halal";
const char* password ="Modalsedikit123";
char mqtt_server[] ="broker.emqx.io";

char Kode_API[] = "AIzaSyA91qoqnAtMUPqSFVvtHyCE8pl4mQdVA70";
char Kode_database[] = "https://skripsi-b0083-default-rtdb.asia-southeast1.firebasedatabase.app/";

Firebase firebase(Kode_database);

WiFiClient espClient;
PubSubClient client(espClient);

// Variabel Sensor Warna
int s0 = 4;
int s1 = 5;
int s2 = 2;
int s3 = 0;
int output_TCS = 15;

// Variabel Sensor Hujan
int output_hujan = A0;

// Variabel Sensor Jarak
int picu = 14;
int tangkap = 12;

// Variabel
int buzzer_pompa = 13;

int led=3;

// Variabel nilai
float nilai_merah, nilai_hijau, nilai_biru, rata_warna ;
float nilai_hujan  = 0;
float nilai_jarak  = 0;
float rata_jarak5,rata_hujan5,rata_warna5;
int waktu1=0,waktu2=0,waktu3=0,waktu4=0,waktu5;
float penampung;
int hari,jam,menit,detik;
int batas;
char penampung_huruf [50];
String waktu;
String nomor = "@Fauzan12345678";
String kalimat1 = "Bahaya Bencana Banjir", kalimat2="Banjir";
int pewaktu;
int aktif;
int mode=LOW;
int jeda_led;

void reconnect() {
  while (!client.connected()) {
    Serial.print("Mencoba Menghubungkan Kembali...");
    if (client.connect("mqttx_34cba85d")) {
      Serial.println("Terkoneksi");
    } else {
      Serial.print("Gagal, rc=");
      Serial.print(client.state());
      Serial.println(" Mencoba Kembali");
      delayMicroseconds(100);
    }
  }
}

void tidur(){
  Serial.println("Sistem Tidur 30 Detik");
  ESP.deepSleep(30e6);
}

void setup() {
  // Persiapan PIN GPIO
  Serial.begin(9600);
  pinMode(0, OUTPUT);
  pinMode(2, OUTPUT);
  pinMode(4, OUTPUT);
  pinMode(5, OUTPUT);
  pinMode(3, OUTPUT);
  pinMode(14, OUTPUT);
  pinMode(13, OUTPUT);
  pinMode(16, OUTPUT);
  pinMode(12, INPUT);
  pinMode(output_TCS, INPUT);
  pinMode(output_hujan, INPUT);
 
  // Persiapan WIFI
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print("."); 
  }
  Serial.println("WiFi connected");

  // Persiapan MQTT
  client.setServer(mqtt_server, 1883);

  // Waktu
  timeClient.begin();

  //Persiapan TCS 3200
  digitalWrite(s0, HIGH);
  digitalWrite(s1, LOW);

  Serial.println("Waktu, Merah, Hijau, Biru, Rata-Rata Warna, Curah Hujan, Jarak");}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  pewaktu = millis();
  // Program Sensor Warna
  if ((pewaktu-waktu1) >= 1000){
    waktu1=pewaktu;
    digitalWrite(s2, LOW);
    digitalWrite(s3, LOW);
    nilai_merah  = pulseIn(output_TCS, LOW);
    digitalWrite(s2, HIGH);
    digitalWrite(s3, HIGH);
    nilai_hijau  = pulseIn(output_TCS, LOW);
    digitalWrite(s2, LOW);
    digitalWrite(s3, HIGH);
    nilai_biru = pulseIn(output_TCS, LOW);
    Serial.print(",");
    Serial.print(nilai_merah);
    Serial.print(",");
    Serial.print(nilai_hijau);
    Serial.print(",");
    Serial.print(nilai_biru);
    rata_warna= (nilai_biru + nilai_merah + nilai_hijau) / 3;
    rata_warna5 +=rata_warna;
    Serial.print(",");
    Serial.print(rata_warna);

  // Program Sensor Hujan
    nilai_hujan  = 1024 - analogRead(output_hujan);
    rata_hujan5 += nilai_hujan;
    Serial.print(",");
    Serial.print(nilai_hujan);

  // Program Sensor Jarak
    digitalWrite(picu, HIGH);
    delayMicroseconds(100);
    digitalWrite(picu, LOW);
    delayMicroseconds(100);
    penampung = pulseIn(tangkap, HIGH);
    nilai_jarak = penampung * 0.034 / 2;
    rata_jarak5 += nilai_jarak;
    Serial.print(",");
    Serial.println(nilai_jarak);

   //publish MQTT
    snprintf(penampung_huruf, 50, "%.2f", nilai_jarak );
    client.publish("Skripsi/sensor/jarakf130402xcv", penampung_huruf);
    snprintf(penampung_huruf, 50, "%.2f", nilai_hujan );
    client.publish("Skripsi/sensor/hujanf130402xcv", penampung_huruf);
    snprintf(penampung_huruf, 50, "%.2f", rata_warna );
    client.publish("Skripsi/sensor/warnaf130402xcv", penampung_huruf);
    snprintf(penampung_huruf, 50, "%d", aktif);
    client.publish("Skripsi/sensor/alatf", penampung_huruf);
    }

  //Firebase
 if((pewaktu-waktu2)>=60000){
  waktu2=pewaktu;
  timeClient.update();
  waktu=timeClient.getFormattedTime();
  jam=timeClient.getHours();
  menit=timeClient.getMinutes();
  detik=timeClient.getSeconds();
  hari=timeClient.getDay();
  firebase.setFloat("Data/jarak/"+String(hari)+"-"+String(jam)+":"+String(menit)+":"+String(detik),nilai_jarak );
  firebase.setFloat("Data/hujan/"+String(hari)+"-"+String(jam)+":"+String(menit)+":"+String(detik),nilai_hujan );
  firebase.setFloat("Data/warna/"+String(hari)+"-"+String(jam)+":"+String(menit)+":"+String(detik),rata_warna );
  }

//deepsleep
if((pewaktu- waktu3)>=300000){
  waktu3=pewaktu;
  digitalWrite(15,LOW);
    Serial.println("Sistem Tidur....");
    tidur();
  }

  if ((pewaktu-waktu5)>=5000){
      //callmebot
      waktu5=pewaktu;
      rata_warna5=rata_warna5/5;
      rata_jarak5=rata_jarak5/5;
      rata_hujan5=rata_hujan5/5;
      Serial.print("rata warna=");
      Serial.println(rata_warna5);
      Serial.print("rata jarak=");
      Serial.println(rata_jarak5);
      Serial.print("rata hujan=");
      Serial.println(rata_hujan5);
    if(((649.6/rata_hujan5)<=1)&&((4.9/rata_jarak5)>=1)&&((80.96/rata_warna5)<=1)){
      telegramCall(nomor, kalimat2);
      digitalWrite(led,HIGH);
      digitalWrite(buzzer_pompa, HIGH);
      Serial.println("banjir");
      aktif=1;
    }
    else if(((373.3/rata_hujan5)<=1)&&((8.5/rata_jarak5)>=1)&&((60.5/rata_warna5)<=1)){
      telegramCall(nomor, kalimat1);
      jeda_led=500;
      digitalWrite(buzzer_pompa, HIGH);
      Serial.println("potensi banjir");
      aktif=1;
    }
    else{
      aktif=0;
      jeda_led=0;
      digitalWrite(buzzer_pompa,LOW);
      digitalWrite(led,LOW);
    }
    rata_jarak5=0;
    rata_warna5=0;
    rata_hujan5=0;
  }

    if(jeda_led==500){
      if((pewaktu-waktu4)>=jeda_led){
        waktu4=pewaktu;
          if(mode==LOW){
            mode=HIGH;
            digitalWrite(led,HIGH);
          }
          else{
            mode=LOW;
            digitalWrite(led,LOW);
          }
      }
    }
}


