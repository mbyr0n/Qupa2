# Arquitectura del QUPA nuevo

## Responsabilidades

La Radxa es la computadora de alto nivel: ejecuta ROS 2, captura la cámara, procesa la visión y comunica el robot con la PC. El ESP32-S3 es el controlador de bajo nivel: ejecuta PID, PWM, lectura de encoders, sensores y watchdog.

```text
/cmd_vel ──> qupa_driver ──> UART/USB ──> ESP32 ──> PID ──> motores
                                      ▲
encoders, IMU, IR, batería ──────────┘
                                      │
                                      ▼
                    /odom /imu/data /scan /battery_state
```

La OV5647 se conecta directamente a la Radxa mediante MIPI CSI. La detección propuesta es OpenCV C++: ROI del anillo, máscara de soportes, HSV, `inRange`, apertura/cierre morfológico, contornos, filtros y tracking temporal.

## Comunicación

Para la integración final se prevé UART de 3.3 V, con TX y RX cruzados y tierra común. Durante desarrollo, el USB del ESP32-S3 puede servir para programación y depuración. El protocolo debe ser binario, con cabecera, tipo, longitud, payload y CRC; el ESP32 debe detener motores si deja de recibir comandos dentro del timeout definido.

## Compatibilidad ROS

Se conservarán interfaces estándar para que los algoritmos puedan ejecutarse en simulación o en el robot físico:

| Función | Interfaz |
|---|---|
| Movimiento | `/cmd_vel` (`geometry_msgs/Twist`) |
| Odometría | `/odom` (`nav_msgs/Odometry`) |
| IMU | `/imu/data` (`sensor_msgs/Imu`) |
| Proximidad | `/scan` (`sensor_msgs/LaserScan`) |
| Batería | `/battery_state` (`sensor_msgs/BatteryState`) |
| Cámara | `/camera/image_raw` y detecciones |
