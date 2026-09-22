# Decisiones y pendientes

## Decisiones tomadas

- La Radxa ejecuta Ubuntu Server y ROS 2 Jazzy `ros-base`.
- Los nodos de producción de Radxa y el firmware del ESP32 se escriben en C/C++.
- Python se reserva para calibración, análisis, generación de parámetros y pruebas.
- La cámara va directamente a la Radxa; no se transportan imágenes por el ESP32.
- La comunicación Radxa–ESP32 será UART en la PCB final y USB puede usarse durante desarrollo.
- No se usará RKNN/NPU mientras el problema se resuelva con OpenCV clásico.
- No se habilitarán overlays de UART, SPI o I²C sin confirmar antes el pinout y su impacto en la consola.

## Pendientes

- Registrar la salida real de la prueba de cámara.
- Fijar el UART concreto de la Radxa y documentar sus pines.
- Definir mensajes y frecuencias del protocolo binario.
- Crear y probar `qupa_msgs`.
- Migrar la interfaz ROS del QUPA anterior sin arrastrar sus drivers Raspberry Pi.
- Añadir URDF/Xacro y TF del chasis nuevo.
