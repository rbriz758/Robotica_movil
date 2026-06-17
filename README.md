# Eurobot 2026 - Sistema Autónomo de Navegación y Manipulación

<p align="center">
  <img src="https://img.shields.io/badge/ROS%202-22314E?style=for-the-badge&logo=ros&logoColor=white" alt="ROS 2" />
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python" />
  <img src="https://img.shields.io/badge/OpenCV-5C3EE8?style=for-the-badge&logo=opencv&logoColor=white" alt="OpenCV" />
  <img src="https://img.shields.io/badge/Arduino-00979D?style=for-the-badge&logo=Arduino&logoColor=white" alt="Arduino" />
  <img src="https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white" alt="Ubuntu" />
</p>

Este repositorio contiene el código fuente y la arquitectura de control para un robot autónomo diseñado para la competición **Eurobot**. El proyecto integra navegación visual (siguelíneas), visión artificial para detección de objetivos, y manipulación física mediante un brazo robótico, todo orquestado bajo **ROS 2** y una **Máquina de Estados Finitos (FSM)**.

## Descripción del Proyecto

El objetivo principal de este proyecto es dotar a una plataforma móvil diferencial de autonomía completa para completar una misión estructurada que consiste en:
1. **Navegar** por un circuito de competición siguiendo una línea negra sobre un fondo claro.
2. **Detectar e identificar** un bloque específico mediante la lectura de marcadores **ArUco**.
3. **Alinearse y recoger** el bloque de manera precisa usando un brazo robótico equipado con una ventosa neumática.
4. **Continuar la navegación**, identificando intersecciones en el camino (cruces en "T").
5. **Identificar visualmente** la zona de almacenamiento final (almacén de color verde).
6. **Depositar** el bloque en la zona designada y concluir la misión.

## Arquitectura y Stack Tecnológico

El cerebro del robot está implementado en el script principal de la FSM, desarrollado en **Python** sobre el middleware **ROS 2**.

- **ROS 2**: Gestión de la concurrencia, ciclo de vida del robot y publicación de trayectorias articulares (`/arm_controller/joint_trajectory`) para el movimiento del brazo.
- **OpenCV (Visión Artificial)**: El pipeline procesa los fotogramas de la cámara cenital en tiempo real para:
  - Binarización, filtrado morfológico y cálculo de momentos para el **seguimiento de línea** con control proporcional.
  - Detección de marcadores empleando `cv2.aruco.ArucoDetector`.
  - Filtrado geométrico y de color (espacio HSV) para la detección de cruces y zonas de almacenamiento.
- **Hardware Integrado**:
  - **Microcontrolador (Arduino)**: Actúa como puente serial (`/dev/ttyACM0`) para inyectar velocidades a los motores DC, controlar el relé de la ventosa neumática y transmitir los datos de la IMU.
  - **IMU (BNO055)**: Sensor inercial utilizado para medir la guiñada (yaw) en tiempo real y asegurar giros exactos.
  - **LiDAR (YDLidar 4ROS)**: Escáner láser 2D para detección de obstáculos y prevención de colisiones.
  - **Brazo Robótico**: Controlado por serial a través de controladores USB (`/dev/ttyUSB0`, `/dev/ttyUSB1`).
  - **Cámara V4L2**: Sensor principal para la percepción reactiva y seguimiento de línea.

## Máquina de Estados Finitos (FSM)

El comportamiento del robot está estructurado de manera determinista usando una FSM con las siguientes fases:

1. **`WAIT_START` & `INIT`**: Espera la confirmación humana de inicio, configura el hardware, coloca el brazo en posición de no oclusión y purga el sensor de la cámara para estabilizar el balance de blancos.
2. **`SEARCH_ARUCO`**: El robot ejecuta su algoritmo de siguelíneas continuo hasta que detecta en el frame completo un marcador ArUco con el ID objetivo (ej. ID 36).
3. **`ALIGN_GRAB`**: Una vez detectado el ArUco, el robot utiliza el área del contorno del marcador en la imagen para calcular la distancia y retroceder o avanzar hasta la posición de agarre perfecta.
4. **`GRAB`**: Secuencia pre-calculada. El brazo desciende a las coordenadas articulares objetivo, activa la bomba de vacío (ventosa), y vuelve a subir asegurando la pieza.
5. **`SEARCH_INTERSECTION`**: Se reanuda el siguelíneas (adaptando las velocidades para el transporte de carga). En paralelo, analiza un recorte geométrico de la derecha de la imagen para detectar una intersección perpendicular válida.
6. **`TURN_RIGHT`**: Ejecuta una maniobra de giro diferencial a la derecha. Utiliza la retroalimentación de la **IMU (BNO055)** para calcular exactamente una rotación relativa de 90 grados y re-encontrar la línea central.
7. **`SEARCH_GREEN`**: Continúa navegando mientras aísla el espectro de color verde. Requiere múltiples frames de confirmación (`green_seen_count`) para asegurar que es la zona de descarga y no ruido del entorno.
8. **`DROP` & `FINISHED`**: El robot avanza dentro de la zona de almacenamiento, desciende el brazo, apaga la ventosa liberando la presión, devuelve el brazo a una posición segura y detiene todos los sistemas.

## Innovaciones Técnicas del Sistema

- **Escudo Anti-Colisiones (LiDAR):** Integración de un YDLidar 4ROS que configura un "túnel virtual asimétrico". El sistema lee datos de `/scan` de forma asíncrona e interrumpe la FSM para frenar los motores instantáneamente si detecta un robot rival en la trayectoria.
- **Odometría Inercial (IMU):** En lugar de depender de encoders en las ruedas (propensos al deslizamiento), los giros de precisión en intersecciones se realizan comprobando en bucle cerrado la orientación (Yaw) proporcionada por la IMU.
- **Inmunidad al Ruido Visual:** Para evitar que el siguelíneas se desvíe por decoraciones oscuras o elementos del tablero, se utiliza el canal de Saturación (del espacio HSV) para enmascarar colores puros y centrarse estrictamente en el trazo negro.
- **Alineamiento Basado en Área:** Se prescinde de cálculos complejos de pose 3D (PnP) para el acercamiento al ArUco, usando en su lugar un control de área de píxeles (`ALIGN_TARGET_AREA`) como estimador de profundidad.

## Ejecución (One-Click Launch)

El proyecto incluye un archivo *launch* de ROS 2 que inicializa todo el hardware y software simultáneamente.

```bash
# 1. Dar permisos a los controladores físicos
sudo chmod 777 /dev/ttyACM0 /dev/ttyUSB0 /dev/ttyUSB1
sudo ln -sf /dev/ttyUSB0 /dev/ttyUSB1

# 2. Cargar los workspaces (Chasis, Brazo y Proyecto)
source ~/robot_ws/install/setup.bash
source ~/brazo_ws/install/setup.bash
source install/setup.bash

# 3. Lanzar la demostración
ros2 launch sprint8_eurobot sprint8_launch.py
```

Una vez ejecutado, espera el mensaje de inicialización en la terminal, verifica que las ventanas de debug visual (OpenCV) aparecen, posiciona el robot y **presiona ENTER** para iniciar la demostración autónoma.


