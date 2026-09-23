# Configuración de la Radxa Zero 3W para QUPA

Guía reproducible para configurar una Radxa Zero 3W como computadora principal del robot QUPA.

La Radxa ejecuta Ubuntu Server y ROS 2; el ESP32-S3 se encarga del control de bajo nivel, y la PC se utiliza para visualización, calibración y análisis.

## Arquitectura objetivo

```text
PC / laptop
  ├─ ROS 2 Desktop, RViz/Foxglove
  ├─ calibraciones y análisis en Python
  └─ teleoperación y logs
          │ Wi-Fi / DDS
          ▼
Radxa Zero 3W
  ├─ Ubuntu 24.04 Server
  ├─ ROS 2 Jazzy + CycloneDDS
  ├─ nodos C++: driver, cámara y visión
  └─ OV5647 por MIPI CSI
          │ UART 3.3 V o USB durante desarrollo
          ▼
ESP32-S3
  ├─ motores, PID y encoders
  ├─ IMU y sensores de proximidad
  ├─ batería y LEDs
  └─ watchdog de seguridad
```

La PC no forma parte del control crítico: la Radxa y el ESP32 deben mantener el robot operativo aunque la laptop se desconecte.

## Estado documentado

### Comprobado en la Radxa

- Ubuntu 24.04.5 LTS, arquitectura ARM64.
- Kernel `6.1.0-1025-rockchip`.
- Wi-Fi funcionando mediante `wlan0`.
- Hostname `qupa2`.
- Zona horaria `America/Guayaquil`.
- NTP sincronizado y `systemd-timesyncd` activo.
- Usuario `ubuntu` con acceso a `sudo` y `video`.
- OpenCV 4.6.0, V4L2 y libcamera tools instalados.
- ROS 2 Jazzy, `colcon`, GCC 13.3 y CMake 3.28 disponibles.
- Workspace creado en `~/qupa_ws`.
- Entorno QUPA creado en `~/qupa_env.bash`.
- Driver RKNPU integrado en el kernel y overlay NPU disponible; la NPU no es necesaria para la detección clásica con OpenCV.

### Pendiente o dependiente del hardware

- Probar físicamente la OV5647 y elegir el pipeline `libcamera`/V4L2.
- Definir y probar el UART del header que no sea la consola de depuración.
- Integrar el protocolo binario Radxa–ESP32 con CRC y watchdog.
- Crear los paquetes ROS 2 del QUPA nuevo.
- Actualizar URDF, TF, parámetros y dimensiones del robot.

## Orden recomendado

1. Instalar Ubuntu Server y actualizar el sistema.
2. Configurar identidad, red, SSH, zona horaria y NTP(opcional).
3. Instalar herramientas de desarrollo C/C++ y diagnóstico.
4. Configurar memoria comprimida solo si las mediciones la justifican.
5. Instalar ROS 2 Jazzy `ros-base` y CycloneDDS.
6. Crear `~/qupa_ws` y compilar un nodo C++ mínimo.
7. Probar la OV5647 cuando esté disponible.
8. Diseñar el protocolo y conectar la Radxa con el ESP32.
9. Añadir los paquetes de cámara, visión, driver, descripción y bringup.

## Estructura prevista del workspace

```text
qupa_ws/src/
├── qupa_msgs/
├── qupa_driver/
├── qupa_camera/
├── qupa_vision/
├── qupa_description/
├── qupa_bringup/
└── qupa_tools/
```

El código que corre continuamente en el robot será C/C++. Python se reservará para calibración, generación de parámetros, análisis de logs y pruebas.

## Documentación

- [Instalación y configuración base](docs/configuracion-base.md)
- [Configuración de la cámara OV5647](docs/camara-ov5647.md)
- [Arquitectura de software](docs/arquitectura.md)
- [Registro de decisiones y pendientes](docs/decisiones.md)

## Principios

- Hacer una copia de `/etc/default/u-boot` antes de cambiar overlays.
- No habilitar I²C, SPI o UART por adivinación: comprobar el pinout y la consola de cada interfaz.
- No instalar RKNN, YOLO o TensorFlow mientras la visión clásica con OpenCV sea suficiente.
- Mantener la interfaz ROS compatible con la simulación: `/cmd_vel`, `/odom`, `/scan`, `/imu/data`, `/battery_state` y detecciones de cámara.
