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

#define API_KEY ""

#define DATABASE_URL "https://sw02-7ded6.firebaseio.com" 

#define USER_EMAIL "softplay008@gmail.com"
#define USER_PASSWORD "swplay!@#"

FirebaseData fbdo;

FirebaseAuth auth;
FirebaseConfig config;

#include <SimpleTimer.h>

SimpleTimer t1;
SimpleTimer t2;


#include <MFRC522.h>
#include <SPI.h>

#define SS_PIN D8
#define RST_PIN D3

MFRC522 rfid(SS_PIN, RST_PIN);

char str[15];

String tag_id;

void setup()
{

  Serial.begin(115200);
  
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

  t1.setInterval(1000,fn1);
  t2.setInterval(1000,fn2);

  SPI.begin();
  rfid.PCD_Init();
}

void loop()
{
  t1.run();
  t2.run();
}

void fn1(void) {
  
  if ( ! rfid.PICC_IsNewCardPresent())
    return;
    
  if ( ! rfid.PICC_ReadCardSerial())
    return;

  String content= "";
  byte letter;
  
  for (byte i = 0; i < rfid.uid.size; i++) {
    content.concat(String(rfid.uid.uidByte[i] < 0x10 ? "0" : ""));
    content.concat(String(rfid.uid.uidByte[i], HEX));
  }
  
  content.toUpperCase();        // 대문자로 변경

  Serial.print("UID tag : ");   // 화면에 출력
  Serial.println(content);      // nfc uid

  // 스캔된 태그 uuid 아이디에 해당하는 umbrella_uidf를 가져온다. 
  tag_id = Firebase.getString(fbdo, "umbrella_uid/" + content) ? fbdo.to<const char *>() : fbdo.errorReason().c_str();
  Serial.print("tag_id : ");
  Serial.println(tag_id);

  String borrow_date = "umbrella_list/" + tag_id + "/borrow_date";         // aNOb
  String borrow_state = "umbrella_list/" + tag_id + "/borrow_state";       // aNb
  String user_number = "umbrella_list/" + tag_id + "/user_number";         // aNOb

  Firebase.setString(fbdo, borrow_date, "aNOb");
  Firebase.setString(fbdo, borrow_state, "aNb");
  Firebase.setString(fbdo, user_number, "aNOb");
}

void fn2(void) {
  
}
```
