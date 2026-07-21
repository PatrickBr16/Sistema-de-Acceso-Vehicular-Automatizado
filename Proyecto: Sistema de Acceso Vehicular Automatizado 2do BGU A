#include <LiquidCrystal_I2C.h>
#include <Servo.h>
#include <Wire.h>

// C++ code

// Definición de Pines de acuerdo a tu solicitud
const int pinTrigger = 2;  // Pin Trigger del sensor ultrasónico asignado al pin 2
const int pinEcho = 3;     // Pin Echo del sensor ultrasónico asignado al pin 3
const int pinLed = 7;     // Salida digital para la luz indicadora parpadeante
const int pinServo = 9;    // Salida PWM para el servomotor de la barra
const int pinBuzzer = 8;   // Salida digital para las alertas sonoras (Buzzer)

// Instancias de los Actuadores e Interfaces
Servo barraVehicular;
LiquidCrystal_I2C lcd(0x27, 16, 2); // Inicializa LCD en dirección 0x27 con 16 columnas y 2 filas

// Variables de Estado del Sistema
int estadoBarrera = 0;      // 0: Cerrada, 1: Subiendo, 2: Abierta (Espera), 3: Bajando 
int posicionServo = 0;     // Ángulo actual del servo (0 grados = Abajo) 
bool estadoLed = LOW;      // Estado actual de la luz y el sonido (encendido/apagado) 

// Variables de Control Externo (HMI)
char modoControl = 'A';      // 'A' = Automático, 'M' = Manual
char comandoManual = ' ';    // 'O' = Open, 'C' = Close, 'S' = Stop/Hold

// Variables de Control para evitar refrescos innecesarios en el LCD (Optimización)
int estadoAnteriorBarrera = -1; 

// Variables de Tiempo (Estrategia No Bloqueante con millis)
unsigned long tiempoAnteriorLed = 0;     
unsigned long tiempoAnteriorServo = 0;   
unsigned long tiempoAnteriorEspera = 0;  

// Constantes de Configuración de Tiempos (en milisegundos)
const long intervaloLed = 250;     // Velocidad del parpadeo y pitido (Cada 250ms cambia) 
const long intervaloServo = 20;    // Velocidad de movimiento de la barra (un grado cada 20ms) 
const long tiempoEsperaAuto = 4000; // Tiempo que la barra permanece abierta (4 segundos) 

// Variable global para almacenar la distancia medida
long distancia = 0;

void setup() {
  // Configuración de pines del sensor ultrasónico
  pinMode(pinTrigger, OUTPUT); // El pin Trigger emite el pulso ultrasónico
  pinMode(pinEcho, INPUT);     // El pin Echo recibe el eco de retorno
  
  pinMode(pinLed, OUTPUT);        
  pinMode(pinBuzzer, OUTPUT);     // Configura el pin del Buzzer como salida
  
  // Inicialización de la Pantalla LCD
  lcd.init();
  lcd.backlight();                // Enciende la luz de fondo del LCD
  
  barraVehicular.attach(pinServo);     
  barraVehicular.write(posicionServo); // Inicializa la barra abajo (0°) 
  digitalWrite(pinLed, LOW);           // Luz inicialmente apagada 
  noTone(pinBuzzer);                  // Asegura que el buzzer inicie en silencio
  Serial.begin(9600); // Abre el monitor serie
}

void loop() {
  unsigned long tiempoActual = millis();            

  // =========================================================================
  // LECTURA DE COMANDOS DESDE EL PUERTO SERIE (HMI)
  // =========================================================================
  if (Serial.available() > 0) {
    char rc = Serial.read();
    if (rc != '\n' && rc != '\r') {
      if (rc == 'A') {
        modoControl = 'A';
      } else if (rc == 'M') {
        modoControl = 'M';
      } else if (rc == 'O' || rc == 'C' || rc == 'S') {
        modoControl = 'M'; // Cualquier comando de acción fuerza el modo manual
        comandoManual = rc;
      }
    }
  }

  // =========================================================================
  // LECTURA ASÍNCRONA/RÁPIDA DEL SENSOR ULTRASÓNICO HC-SR04
  // =========================================================================
  digitalWrite(pinTrigger, LOW);
  delayMicroseconds(2);
  digitalWrite(pinTrigger, HIGH);
  delayMicroseconds(10);
  digitalWrite(pinTrigger, LOW);
  
  long duracion = pulseIn(pinEcho, HIGH);
  distancia = duracion * 0.034 / 2; // Cálculo matemático para obtener distancia en cm

  // Comentado para evitar conflicto con la telemetría JSON de la interfaz HMI
  // Serial.print("Distancia detectada: ");
  // Serial.print(distancia);
  // Serial.println(" cm");
  
  // Definición del umbral: Detecta vehículo si la distancia está entre 1 y 20 centímetros
  bool deteccionVehiculo = (distancia > 0 && distancia < 20);

  // =========================================================================
  // GESTIÓN DINÁMICA DE LA INTERFAZ HMI (PANTALLA LCD)
  // Actualiza el texto solo cuando hay un cambio de estado para evitar parpadeos
  // =========================================================================
  if (estadoBarrera != estadoAnteriorBarrera) {
    lcd.clear();
    switch (estadoBarrera) {
      case 0:
        lcd.setCursor(0, 0); lcd.print("BARRERA: CERRADA");
        lcd.setCursor(0, 1); lcd.print("   PROHIBIDO!   ");
        break;
      case 1:
        lcd.setCursor(0, 0); lcd.print("  BARRERA MOV.  ");
        lcd.setCursor(0, 1); lcd.print("  --ABRIENDO--  ");
        break;
      case 2:
        lcd.setCursor(0, 0); lcd.print("  ACCESO LIBRE   ");
        lcd.setCursor(0, 1); lcd.print(" SIGA ADELANTE! ");
        break;
      case 3:
        lcd.setCursor(0, 0); lcd.print(" BARRERA: MOV.  ");
        lcd.setCursor(0, 1); lcd.print("  --CERRANDO--  ");
        break;
    }
    estadoAnteriorBarrera = estadoBarrera; // Guarda el estado actual
  }

  // =========================================================================
  // MÁQUINA DE ESTADOS PARA EL CONTROL DE LA BARRERA Y ADVERTENCIAS
  // =========================================================================
  switch (estadoBarrera) { 
    
    case 0: // ESTADO: REPOSO (BARRERA CERRADA)
      digitalWrite(pinLed, LOW);   // Asegura luz apagada 
      noTone(pinBuzzer);           // Asegura sonido apagado
      
      if (modoControl == 'A') {
        if (deteccionVehiculo == true) { // Si el sensor ultrasónico detecta el carro cerca
          estadoBarrera = 1;         // Pasa al estado Subiendo 
          tiempoAnteriorServo = tiempoActual; 
          tiempoAnteriorLed = tiempoActual;   
        }
      } else { // Modo Manual
        if (comandoManual == 'O') {
          estadoBarrera = 1;         // Pasa al estado Subiendo
          tiempoAnteriorServo = tiempoActual; 
          tiempoAnteriorLed = tiempoActual;   
          comandoManual = ' ';       // Consumir comando
        }
      }
      break;

    case 1: // ESTADO: SUBIENDO LA BARRA
      // Interrupción manual desde la interfaz
      if (modoControl == 'M') {
        if (comandoManual == 'C') {
          estadoBarrera = 3;         // Forzar cierre
          tiempoAnteriorServo = tiempoActual;
          comandoManual = ' ';
          break;
        } else if (comandoManual == 'S') {
          // Detener (se mantiene el estado 1 pero sin avanzar el servo)
          break;
        }
      }

      // Control Simultáneo de Parpadeo (LED) y Pitido Intermitente (Buzzer)
      if (tiempoActual - tiempoAnteriorLed >= intervaloLed) { 
        tiempoAnteriorLed = tiempoActual;                     
        estadoLed = !estadoLed;                               
        digitalWrite(pinLed, estadoLed);                     
        
        if (estadoLed == HIGH) {
          tone(pinBuzzer, 1500);  // Emite un tono agudo de 1500 Hz
        } else {
          noTone(pinBuzzer);      // Apaga el tono
        }
      }
      
      // Control Movimiento Gradual de la Barra 
      if (tiempoActual - tiempoAnteriorServo >= intervaloServo) { 
        tiempoAnteriorServo = tiempoActual;                       
        if (posicionServo < 90) { 
          posicionServo++;        
          barraVehicular.write(posicionServo); 
        } else {
          estadoBarrera = 2; // Llegó a 90°, pasa a estado de Espera 
          tiempoAnteriorEspera = tiempoActual; // Guarda el tiempo de apertura 
          if (modoControl == 'M') comandoManual = ' '; // Consumir comando de apertura al completarse
        }
      }
      break;

    case 2: // ESTADO: ESPERA (BARRA ABIERTA)
      // La luz y el sonido continúan en intervalos
      if (tiempoActual - tiempoAnteriorLed >= intervaloLed) { 
        tiempoAnteriorLed = tiempoActual;                      
        estadoLed = !estadoLed;                                
        digitalWrite(pinLed, estadoLed);                      
        
        if (estadoLed == HIGH) {
          tone(pinBuzzer, 1500); 
        } else {
          noTone(pinBuzzer);
        }
      }

      // Transición al estado 3 (Bajar barra)
      if (modoControl == 'A') {
        // En automático: mientras el vehículo esté en rango, mantenemos el temporizador reiniciado.
        if (deteccionVehiculo == true) {
          tiempoAnteriorEspera = tiempoActual; 
        }
        
        // Se cierra cuando transcurren 2 segundos continuos SIN detectar vehículo (evita cierres instantáneos)
        if (tiempoActual - tiempoAnteriorEspera >= 2000) { 
          estadoBarrera = 3; 
          tiempoAnteriorServo = tiempoActual; 
        }
      } else {
        // En manual: reacciona al comando de cerrar
        if (comandoManual == 'C') {
          estadoBarrera = 3;
          tiempoAnteriorServo = tiempoActual;
          comandoManual = ' ';
        } else if (comandoManual == 'O') {
          // Si envían "abrir" mientras ya está abierta, reiniciar temporizador/comando
          comandoManual = ' ';
        }
      }
      break;

    case 3: // ESTADO: BAJANDO LA BARRA
      // Interrupción manual desde la interfaz
      if (modoControl == 'M') {
        if (comandoManual == 'O') {
          estadoBarrera = 1;         // Forzar apertura
          tiempoAnteriorServo = tiempoActual;
          comandoManual = ' ';
          break;
        } else if (comandoManual == 'S') {
          // Detener (se mantiene el estado 3 pero sin avanzar el servo)
          break;
        }
      }

      // Mantiene parpadeo visual y sonoro durante el descenso de seguridad
      if (tiempoActual - tiempoAnteriorLed >= intervaloLed) { 
        tiempoAnteriorLed = tiempoActual;                     
        estadoLed = !estadoLed;                               
        digitalWrite(pinLed, estadoLed);                     
        
        if (estadoLed == HIGH) {
          tone(pinBuzzer, 1200);  // Tono ligeramente más bajo para diferenciar descenso
        } else {
          noTone(pinBuzzer);
        }
      }

      // Control Movimiento Gradual de Descenso 
      if (tiempoActual - tiempoAnteriorServo >= intervaloServo) { 
        tiempoAnteriorServo = tiempoActual;                       
        if (posicionServo > 0) { 
          posicionServo--;       
          barraVehicular.write(posicionServo); 
        } else {
          estadoBarrera = 0; // Volvió a 0°, regresa a estado de Reposo 
          if (modoControl == 'M') comandoManual = ' '; // Consumir comando
        }
      }
      break;
  }

  // =========================================================================
  // ENVÍO PERIÓDICO DE TELEMETRÍA JSON (CADA 200MS)
  // =========================================================================
  static unsigned long tiempoAnteriorTelemetry = 0;
  if (tiempoActual - tiempoAnteriorTelemetry >= 200) {
    tiempoAnteriorTelemetry = tiempoActual;
    Serial.print("{\"dist\":");
    Serial.print(distancia);
    Serial.print(",\"state\":");
    Serial.print(estadoBarrera);
    Serial.print(",\"mode\":\"");
    Serial.print(modoControl);
    Serial.print("\",\"pos\":");
    Serial.print(posicionServo);
    Serial.println("}");
  }
}
