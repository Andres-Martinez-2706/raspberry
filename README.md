# venv

¡Claro bro! 😎 Te explico paso a paso cómo llegar a esa ruta en tu Raspberry Pi, activar el entorno virtual y correr el script `main.py` que está en `/home/admin/desktop/venv/sensor/`.

### 🧭 1. Abre la terminal en tu Raspberry Pi

Puedes usar el entorno gráfico o conectarte por SSH, como prefieras.

---

### 📁 2. Navega hasta el entorno virtual

```bash
cd /home/admin/desktop/venv

```

---

### 🐍 3. Activa el entorno virtual

Depende de cómo lo creaste, pero lo más común es:

```bash
source bin/activate

```

Si estás usando Python 3 y `venv`, este comando funciona. Si por algún motivo no, dime y miramos otra forma.

---

### 🚀 4. Ejecuta tu script

Una vez activado el entorno, corre el archivo `main.py` así:

```bash
python sensor/main.py

```

---

### 👀 Extra (opcional): Verifica que estás usando el Python correcto

Puedes verificar que estás usando el del entorno virtual con:

```bash
which python

```

Debería decir algo como:

`/home/admin/desktop/venv/bin/python`

---

# conexión

---

### 🧩 **Componentes necesarios**

- Sensor **MAX30100**
- 2 x **Botones (Pulsadores)**
- 1 x **Buzzer**
- 3 x **Resistencias de 10kΩ** (para los botones, como pull-down)
- Cables macho-macho
- Protoboard

---

### ⚙️ **Conexiones**

### 📟 Sensor MAX30100 (I2C)

| MAX30100 Pin | Raspberry Pi Pin |
| --- | --- |
| VIN | 3.3V (Pin 1) |
| GND | GND (Pin 6) |
| SDA | GPIO2 (Pin 3) |
| SCL | GPIO3 (Pin 5) |

> 🧠 Asegúrate de que el I2C esté habilitado con sudo raspi-config → Interface Options → I2C → Enable
> 

---

### 🔘 Botón de Medición (Inicia lectura)

| Botón Pin | Raspberry Pi Pin |
| --- | --- |
| 1 | GND (Pin 39, por ejemplo) |
| 2 | GPIO17 (Pin 11) |
| + Resistencia de 10kΩ como pull-down entre el pin 2 del botón y GND |  |

---

### 🚨 Botón de Emergencia (Activa buzzer + Telegram)

| Botón Pin | Raspberry Pi Pin |
| --- | --- |
| 1 | GND (Pin 39) |
| 2 | GPIO27 (Pin 13) |
| + Resistencia de 10kΩ como pull-down entre el pin 2 del botón y GND |  |

---

### 🔊 Buzzer

| Buzzer Pin | Raspberry Pi Pin |
| --- | --- |
| + (largo) | GPIO23 (Pin 16) |
| - (corto) | GND (Pin 14) |

> ⚠️ Si el buzzer es activo, suena solo con HIGH. Si es pasivo, puedes generar tonos.
> 

---

### 🖼️ Visual (Texto)

```
        +--------------------+
        |    Raspberry Pi    |
        +--------------------+
               |        |
         SDA --+        +-- SCL
               |        |
            MAX30100    |
               |        |
            VCC---3.3V  |
            GND---GND   |
                       BTN1 (GPIO17) <-- para medición
                       BTN2 (GPIO27) <-- para alarma
                          |
                       BUZZER (GPIO23)

```

---

# CODIGO

---

### ✅ Código Completo Adaptado

```python
import RPi.GPIO as GPIO
import time
import json
import requests
from max30100 import MAX30100
from scipy.signal import find_peaks

# === CONFIGURACIONES ===
BOT_TOKEN = "TU_TOKEN"
CHAT_IDS = [12345678, 87654321]  # IDs de Telegram (puedes leer de un .json si quieres)
BUZZER_PIN = 23
BUTTON_MEASURE_PIN = 17
BUTTON_EMERGENCY_PIN = 27
DURATION = 15  # segundos de medición

# === SETUP GPIO ===
GPIO.setmode(GPIO.BCM)
GPIO.setup(BUZZER_PIN, GPIO.OUT)
GPIO.setup(BUTTON_MEASURE_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
GPIO.setup(BUTTON_EMERGENCY_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

# === FUNCIONES ===
def enviar_telegram(msg):
    for chat_id in CHAT_IDS:
        url = f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage"
        requests.post(url, data={"chat_id": chat_id, "text": msg})

def activar_buzzer(t=1):
    GPIO.output(BUZZER_PIN, GPIO.HIGH)
    time.sleep(t)
    GPIO.output(BUZZER_PIN, GPIO.LOW)

def medir():
    sensor = MAX30100()
    sensor.enable_spo2()
    ir_data = []
    red_data = []

    print("Iniciando medición. ¡No retires el dedo!")
    start = time.time()

    while time.time() - start < DURATION:
        sensor.read_sensor()
        ir_data.append(sensor.ir)
        red_data.append(sensor.red)
        time.sleep(0.1)

    print("Medición finalizada. Puedes retirar el dedo.")
    return ir_data, red_data

def procesar(ir_data):
    # Normalizar para facilitar picos
    base = [v if v > 1000 else 0 for v in ir_data]
    peaks, _ = find_peaks(base, distance=20)

    if len(peaks) > 1:
        dur = DURATION
        pulsos = len(peaks)
        bpm = (pulsos / dur) * 60
    else:
        bpm = 0

    return bpm

def guardar_dato(bpm, spo2):
    entrada = {
        "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
        "bpm": bpm,
        "spo2": spo2
    }
    try:
        with open("mediciones.json", "r+") as f:
            datos = json.load(f)
    except:
        datos = []

    datos.append(entrada)

    with open("mediciones.json", "w") as f:
        json.dump(datos, f, indent=4)

def recomendar(bpm, spo2):
    if bpm < 50 or bpm > 120 or spo2 < 90:
        mensaje = f"⚠️ Alerta: BPM={bpm:.1f}, SpO₂={spo2:.1f}%\nValores anormales detectados. Consulta médica sugerida."
        activar_buzzer(2)
    else:
        mensaje = f"✅ Lectura estable:\nBPM={bpm:.1f}\nSpO₂={spo2:.1f}%"
    print(mensaje)
    enviar_telegram(mensaje)

# === LOOP PRINCIPAL ===
print("Sistema listo. Esperando interacción...")

try:
    while True:
        if GPIO.input(BUTTON_MEASURE_PIN) == GPIO.LOW:
            ir_data, red_data = medir()
            bpm = procesar(ir_data)
            spo2 = 97  # valor fijo simulado (puedes estimarlo si lo deseas)
            guardar_dato(bpm, spo2)
            recomendar(bpm, spo2)
            time.sleep(1)

        if GPIO.input(BUTTON_EMERGENCY_PIN) == GPIO.LOW:
            print("¡Emergencia simulada!")
            activar_buzzer(3)
            enviar_telegram("🚨 Emergencia detectada. ¡Atención urgente!")
            time.sleep(1)

except KeyboardInterrupt:
    GPIO.cleanup()
    print("Programa terminado.")

```

---
