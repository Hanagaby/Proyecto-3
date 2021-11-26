# Proyecto-3

//UNIVERSIDAD DEL VALLE DE GUATEMALA
//BE3015 - ELECTRÓNICA DIGITAL 2
//PROYECTO 2
//HELLEN ANA GABRIELA CASTILLO MÉNDEZ
//19832

#include <Wire.h>
#include "MAX30105.h"
#include "spo2_algorithm.h"

#include <Adafruit_NeoPixel.h>

MAX30105 Sensor;

Adafruit_NeoPixel tira = Adafruit_NeoPixel(1, 12, NEO_GRB + NEO_KHZ800);

#define sens_brillo 255

#if defined(__AVR_ATmega328P__) || defined(__AVR_ATmega168__)
//Arduino Uno doesn't have enough SRAM to store 100 samples of IR led data and red led data in 32-bit format
//To solve this problem, 16-bit MSB of the sampled data will be truncated. Samples become 16-bit data.
uint16_t irBuffer[100]; //infrared LED sensor data
uint16_t redBuffer[100];  //red LED sensor data
#else
uint32_t irBuffer[100]; //infrared LED sensor data
uint32_t redBuffer[100];  //red LED sensor data
#endif

int32_t bufferLength; //data length
int32_t spo2; //SPO2 value
int8_t validSPO2; //indicator to show if the SPO2 calculation is valid
int32_t Frecuencia_cardiaca; //heart rate value
int8_t validFC; //indicator to show if the heart rate calculation is valid

byte pulseLED = 11; //Must be on PWM pin
byte readLED = 13; //Blinks with each data read

//int I2C_SPEED_FAST;
unsigned char Entrada;

void setup()
{
  Serial.begin(115200); // initialize serial communication at 115200 bits per second:
  Serial2.begin(115200); //
  Serial.println("Mediendo Spo2 y FC");

  pinMode(pulseLED, OUTPUT);
  pinMode(readLED, OUTPUT);

  // Initialize sensor
  if (!Sensor.begin(Wire, I2C_SPEED_FAST)) //Use default I2C port, 400kHz speed
  {
    Serial.println(F("MAX30102 no encontrado."));
    while (1);
  }

  Serial.println(F("Coloca el dedo indice en el sensor y presiona enter"));
  while (Serial.available() == 0) ; //wait until user presses a key
  Serial.read();

  byte ledBrightness = 60; //Options: 0=Off to 255=50mA
  byte sampleAverage = 4; //Options: 1, 2, 4, 8, 16, 32
  byte ledMode = 2; //Options: 1 = Red only, 2 = Red + IR, 3 = Red + IR + Green
  byte sampleRate = 100; //Options: 50, 100, 200, 400, 800, 1000, 1600, 3200
  int pulseWidth = 411; //Options: 69, 118, 215, 411
  int adcRange = 4096; //Options: 2048, 4096, 8192, 16384

  Sensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange); //Configure sensor with these settings

  {
    tira.begin();       // inicializacion
    tira.show();        // muestra datos en pixel
  }
}

void loop()
{
  bufferLength = 100; //buffer length of 100 stores 4 seconds of samples running at 25sps

  //read the first 100 samples, and determine the signal range
  for (byte i = 0 ; i < bufferLength ; i++)
  {
    while (Sensor.available() == false) //do we have new data?
      Sensor.check(); //Check the sensor for new data

    redBuffer[i] = Sensor.getRed();
    irBuffer[i] = Sensor.getIR();
    Sensor.nextSample(); //We're finished with this sample so move to next sample

    Serial.print(F("red="));
    Serial.print(redBuffer[i], DEC);
    Serial.print(F(", ir="));
    Serial.println(irBuffer[i], DEC);
  }

  //calculate heart rate and SpO2 after first 100 samples (first 4 seconds of samples)
  maxim_heart_rate_and_oxygen_saturation(irBuffer, bufferLength, redBuffer, &spo2, &validSPO2, &Frecuencia_cardiaca, &validFC);

  //Continuously taking samples from MAX30102.  Heart rate and SpO2 are calculated every 1 second
  while (1)
  {
    //dumping the first 25 sets of samples in the memory and shift the last 75 sets of samples to the top
    for (byte i = 25; i < 100; i++)
    {
      redBuffer[i - 25] = redBuffer[i];
      irBuffer[i - 25] = irBuffer[i];
    }

    //take 25 sets of samples before calculating the heart rate.
    for (byte i = 75; i < 100; i++)
    {
      while (Sensor.available() == false) //do we have new data?
        Sensor.check(); //Check the sensor for new data

      digitalWrite(readLED, !digitalRead(readLED)); //Blink onboard LED with every data read

      redBuffer[i] = Sensor.getRed();
      irBuffer[i] = Sensor.getIR();
      Sensor.nextSample(); //We're finished with this sample so move to next sample

      //send samples and calculation result to terminal program through UART
      Serial.print(F("red="));
      Serial.print(redBuffer[i], DEC);
      Serial.print(F(", ir="));
      Serial.println(irBuffer[i], DEC);
      Serial.println(' ');

      Serial.print(F("FC="));
      Serial.print(Frecuencia_cardiaca, DEC);

      Serial.print(F(", FCvalid="));
      Serial.println(validFC, DEC);
      Serial.println(' ');

      Serial.print(F("SPO2="));
      Serial.print(spo2, DEC);

      Serial.print(F(", SPO2Valid="));
      Serial.println(validSPO2, DEC);
      Serial.println(' ');
    }

    //After gathering 25 new samples recalculate HR and SP02
    maxim_heart_rate_and_oxygen_saturation(irBuffer, bufferLength, redBuffer, &spo2, &validSPO2, &Frecuencia_cardiaca, &validFC);

    tira.setBrightness(40);               // brillo global para toda la tira
    //for (int i = 0; i < 8; i++) { // bucle para recorrer posiciones 0 a 7
    tira.setPixelColor(0, 0, 0, 255);   // cada pixel en color azul (posicion,R,G,B)
    tira.show();      // muestra datos en pixel.
    tira.clear(); //apagar neopixel
    tira.setPixelColor(0, 0, 255, 0);   // cada pixel en color azul (posicion,R,G,B)
    tira.show();      // muestra datos en pixel
    tira.clear(); //apagar neopixel
    tira.setPixelColor(0, 255, 0, 0);   // cada pixel en color azul (posicion,R,G,B)
    tira.show();      // muestra datos en pixel
    tira.clear(); //apagar neopixel
    //delay(500);       // breve demora de medio segundo
    //tira.setPixelColor(0, 0, 0, 0); // apaga el pixel
    //tira.show();      // muestra datos en pixel
    //}

  }
  if (Serial2.available() > 0) {
    Entrada = Serial2.read ();
    if (Entrada == 'E') {
      int Temporal1;
      int Temporal2;
      Temporal1 = Frecuencia_cardiaca;
      Temporal2 = spo2;
      Serial2.print('R');
      Serial2.print(Temporal1);
      Serial.println(Temporal1);
      Serial2.print(Temporal2);
      Serial.println(Temporal2);
      Serial2.print('F');
      Entrada = 0;

    }
  }
}
