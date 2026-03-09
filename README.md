import network
import time
import socket
import machine
import json

# Definir constantes del TCS34725 que faltaban
ENABLE = 0x00
ATIME = 0x01
CONTROL = 0x0F
CDATA = 0x14

i2c = machine.I2C(0, scl=machine.Pin(22), sda=machine.Pin(21), freq=400000)
TCS34725_ADDR = 0x29
CMD = 0x80

def init_tcs34725():
    i2c.writeto_mem(TCS34725_ADDR, CMD | ENABLE, b'\x03')
    i2c.writeto_mem(TCS34725_ADDR, CMD | ATIME, b'\xEB')
    i2c.writeto_mem(TCS34725_ADDR, CMD | CONTROL, b'\x01')
    time.sleep(0.5)

def get_datetime_formatted():
    t = time.localtime()
    year, month, day, hour, minute, second = t[0:6]
    return f"{day:02d}/{month:02d}/{year} {hour:02d}:{minute:02d}:{second:02d}"

# Función para mostrar menú
def show_menu():
    print("\n" + "=" * 40)
    print("       MENU PRINCIPAL")
    print("=" * 40)
    print(f"Fecha y Hora: {get_datetime_formatted()}")
    print("\nOpciones:")
    print("1. Ver estado de conexión")
    print("2. Ver información del ESP32")
    print("3. Continuar enviando datos")
    print("=" * 40)

    try:
        data = i2c.readfrom_mem(TCS34725_ADDR, CMD | CDATA, 8)

        luminicidad = (data[1] << 8) | data[0]
        rojo = (data[3] << 8) | data[2]
        verde = (data[5] << 8) | data[4]
        azul = (data[7] << 8) | data[6]

        print(f"Lectura TCS34725 -> R:{rojo} G:{verde} B:{azul} C:{luminicidad}")
        return rojo, verde, azul, luminicidad

    except Exception as e:
        print("Error al leer TCS34725:", e)
        return None

# Inicializar sensor
init_tcs34725()
print("Escaneo I2C:", [hex(x) for x in i2c.scan()])

# Conectar a WiFi
wf = network.WLAN(network.STA_IF)
wf.active(True)

print("Conectando a WiFi...")
wf.connect('Nancy', '39798969')

while not wf.isconnected():
    print(".")
    time.sleep(1)

print("Conectado a:", wf.ifconfig())

show_menu()  # Mostrar menú al conectarse

# Crear socket cliente para conectarse a la PC
print("Conectando al servidor en la PC...")
s_pc = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

print("Conectando a la Raspberry Pi...")
s_pi = socket.socket(socket.AF_INET, socket.SOCK_STREAM)#192.168.1.10

try:
    # conectar a la PC (ajusta IP y puerto según tu servidor)
    s_pc.connect(('192.168.1.8', 2024))
    print("Conectado al PC")

    s_pi.connect(('192.168.1.10', 2024))
    print("Conectado a la Raspberry Pi")
    packet_id = 0   # contador de paquetes
    # enviar datos cada segundo
    while True:
        rgb_data = show_menu()# Usa show_menu para leer sensor
        if rgb_data is not None:
            rojo, azul, verde, luminicidad = rgb_data
            
            # crear estructura JSON con timestamp y ID de paquete
            payload = {
                "packet_id": packet_id,
                "timestamp": int(time.time()),
                "rojo": rojo,
                "azul": azul,
                "verde": verde,
                "luminicidad": luminicidad
            }
            
            # convertir a JSON y enviar
            mensaje = json.dumps(payload) + "\n"
            try:
                s_pc.send(mensaje.encode())
            except:
                print("Error enviando al PC")

            try:
                s_pi.send(mensaje.encode())
            except:
                print("Error enviando a la Raspberry")
        else:
            print("No se pudo leer el sensor")
            
        packet_id += 1
        
        time.sleep(1)  # esperar 1 segundo antes de enviar el siguiente dato
        
except Exception as e:
    print(f"Error de conexión: {e}")
finally:
    s_pc.close()
    s_pi.close()
    print("Socket cerrado")


