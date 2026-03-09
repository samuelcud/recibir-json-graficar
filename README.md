# -*- coding: utf-8 -*-
"""
Created on Thu Mar  5 22:48:49 2026

@author: smuel
"""

#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import socket
import json
import sys
import matplotlib.pyplot as plt

HOST = '0.0.0.0'
PORT = 2024

def main():
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    try:
        server_socket.bind((HOST, PORT))
        server_socket.listen(1)

        print("=" * 60)
        print("   SERVIDOR + GRÁFICA EN TIEMPO REAL TCS34725")
        print("=" * 60)
        print(f"Escuchando en puerto {PORT}...")
        print("Esperando conexión del ESP32...\n")

        client_socket, client_address = server_socket.accept()
        print(f"Cliente conectado desde: {client_address[0]}:{client_address[1]}")
        print("-" * 60)

        plt.ion()
        fig, ax = plt.subplots()
        ax.set_title("Lecturas TCS34725 en tiempo real")
        ax.set_xlabel("Paquete")
        ax.set_ylabel("Valor")
        ax.grid(True)

        x_data = []
        rojo_data = []
        verde_data = []
        azul_data = []
        luminicidad_data = []

        # AQUÍ ESTÁ EL ÚNICO CAMBIO: colores reales para cada línea
        line_rojo, = ax.plot([], [], 'red', label='Rojo', linewidth=2)
        line_verde, = ax.plot([], [], 'green', label='Verde', linewidth=2)
        line_azul, = ax.plot([], [], 'blue', label='Azul', linewidth=2)
        line_lum, = ax.plot([], [], 'gray', label='Luminicidad', linewidth=2)

        ax.legend()

        buffer = ""

        try:
            while True:
                data = client_socket.recv(1024).decode("utf-8")

                if not data:
                    print("\nConexión cerrada por el cliente")
                    break

                buffer += data

                while "\n" in buffer:
                    line, buffer = buffer.split("\n", 1)

                    if line.strip():
                        try:
                            payload = json.loads(line)

                            packet_id = payload.get("packet_id", 0)
                            rojo = payload.get("rojo", 0)
                            verde = payload.get("verde", 0)
                            azul = payload.get("azul", 0)
                            luminicidad = payload.get("luminicidad", 0)

                            print(f"Paquete {packet_id} -> R:{rojo} G:{verde} B:{azul} C:{luminicidad}")

                            x_data.append(packet_id)
                            rojo_data.append(rojo)
                            verde_data.append(verde)
                            azul_data.append(azul)
                            luminicidad_data.append(luminicidad)

                            line_rojo.set_data(x_data, rojo_data)
                            line_verde.set_data(x_data, verde_data)
                            line_azul.set_data(x_data, azul_data)
                            line_lum.set_data(x_data, luminicidad_data)

                            ax.relim()
                            ax.autoscale_view()

                            fig.canvas.draw()
                            fig.canvas.flush_events()

                        except json.JSONDecodeError as e:
                            print(f"Error al parsear JSON: {e}")
                            print(f"Datos recibidos: {line}\n")

        except KeyboardInterrupt:
            print("\nServidor interrumpido por el usuario")

        finally:
            client_socket.close()

    except OSError as e:
        print(f"Error al vincular el puerto {PORT}")
        print(f"Detalles: {e}")
        print("Asegúrate de que el puerto no esté en uso")
        sys.exit(1)

    finally:
        server_socket.close()
        print("Servidor cerrado")

if __name__ == "__main__":
    main()
