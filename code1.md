```c++
#if defined(ESP32)
#include <WiFi.h>
#include <FirebaseESP32.h>
#elif defined(ESP8266)
#include <ESP8266WiFi.h>
#include <FirebaseESP8266.h>
#endif


#include <addons/TokenHelper.h>
#include <addons/RTDBHelper.h>

#define WIFI_SSID "soft"
#define WIFI_PASSWORD "ss2420052"

#define API_KEY "AIzaSyCWJ3HGPA6KrG9zhm_eYbFroM7NNs1wjeo"
#define DATABASE_URL "https://sw02-7ded6.firebaseio.com" //<databaseName>.firebaseio.com or <databaseName>.<region>.firebasedatabase.app

#define USER_EMAIL "softplay008@gmail.com"
#define USER_PASSWORD "swplay!@#"

FirebaseData fbdo;    // 파이어베이스 변수(객체)

FirebaseAuth auth;     
FirebaseConfig config;

#include <SoftwareSerial.h>
#include <DFPlayer_Mini_Mp3.h>

#include <SimpleTimer.h>

SimpleTimer t1;
SimpleTimer t2;

int trig = 14;      // 송신을 D5번 핀으로 설정
int echo = 12;      // 수신을 D6번 핀으로 설정

String gate_state = "";
String mp3_onoff = "OFF";

void setup()
{
  
  Serial.begin(9600);

  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Connecting to Wi-Fi");
  while (WiFi.status() != WL_CONNECTED)
  {
    Serial.print(".");
    delay(300);
  }
  Serial.println();
  Serial.print("Connected with IP: ");
  Serial.println(WiFi.localIP());
  Serial.println();

  Serial.printf("Firebase Client v%s\n\n", FIREBASE_CLIENT_VERSION);

  config.api_key = API_KEY;

  auth.user.email = USER_EMAIL;
  auth.user.password = USER_PASSWORD;

  config.database_url = DATABASE_URL;

  config.token_status_callback = tokenStatusCallback; //see addons/TokenHelper.h

  Firebase.begin(&config, &auth);

  Firebase.reconnectWiFi(true);

  
  pinMode(trig, OUTPUT);    // 송신으로 연결된 핀을 OUTPUT으로 설정
  pinMode(echo, INPUT);     // 수신으로 연결된 핀을 INPUT으로 설정

  mp3_set_serial(Serial);   // mp3 셋팅
  delay(1);                 
  mp3_set_volume(30);       // 소리 30
  
  t1.setInterval(1000,fn1); // fn1 함수 1초에 한번씩 실행 설정
  t2.setInterval(1000,fn2); // fn2 함수 1초에 한번씩 실행 설정
}

void loop()
{
  t1.run();   // fn1 함수 시작
  t2.run();   // fn2 함수 시작
}

void fn1(void) {
  digitalWrite(trig, HIGH);     // 초음파센서 신호 발사
  delayMicroseconds(10);        // 딜레이
  digitalWrite(trig, LOW);      // 초음파 신호 정지

  // distance 변수에 거리값이 저장된다. 예를 들면 10, 30, 98...
  int distance = pulseIn(echo, HIGH) * 17 / 1000;

  // 측정된 거리 값를 시리얼 모니터에 출력합니다.
  Serial.print("거리 : ");
  Serial.print(distance);
  Serial.println(" cm");

  // 파이어베이스에 읽어온 값이 Y이면 mp3_onoff 변수의 값을 OFF로 저장한다.
  if(gate_state=="Y") {
    mp3_onoff = "OFF"; 
  }
  
  if(distance<30) {       // 거리가 30미만이면(게이트에 사람이 있으면) 
    if(gate_state=="N") { // gate_state값이 N이면 인증을 안했다는 의미. Y이면 인증을 했다는 의미
      Serial.println("인증을 하지 않고 들어왔다. mp3_onoff를 ON으로 설정"); 
      mp3_onoff = "ON";   // mp3_onoff 값을 ON으로 저장한다.
    }
  }

  if(mp3_onoff=="ON") {   // mp3_onoff 값이 ON이면 mp3 작동
    Serial.println("MP3를 울립니다.");
    mp3_play(1);
    delay(2000);    
  } else if(mp3_onoff=="OFF") { // mp3_onoff 값이 OFF이면 mp3 끄기
    Serial.println("MP3를 끕니다");
    mp3_stop();     
  }
  
  delay(1000);
}

void fn2(void) {
  // 파이어베이스의 umbrella_gate/gate_state 의 값을 가져온다. aNb 또는 aYb가 되는데
  // N, Y를 얻기위해서 aNb에서 a+1 부터 b-1까지 잘라서 gate_state 변수에 넣는다.
  gate_state = Firebase.getString(fbdo, "umbrella_gate/gate_state") ? fbdo.to<const char *>() : fbdo.errorReason().c_str();    
  gate_state = gate_state.substring(gate_state.indexOf("a")+1, gate_state.indexOf("b"));

  // 결과 출력
  Serial.print("gate_state: ");
  Serial.println(gate_state);   // N

}
```
