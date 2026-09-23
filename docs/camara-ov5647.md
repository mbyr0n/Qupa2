# Configuración de la cámara OV5647

Esta página documenta la puesta en marcha de la Raspberry Pi Camera V1.3 / OV5647 en la Radxa Zero 3W con Ubuntu Server. La cámara observa un espejo catadióptrico para obtener una vista de 360 grados del entorno.

La guía está escrita para repetir la configuración en varios robots. Los comandos se ejecutan en la Radxa por SSH, salvo los pasos marcados como **PC**. Los valores de resolución, exposición y calibración que aparecen aquí corresponden a la unidad utilizada durante las pruebas; si se cambia la cámara, el cable, el espejo o la iluminación, hay que verificar y posiblemente recalibrar.

## Resultado esperado

Al terminar esta guía, la Radxa debe poder:

```text
OV5647 → MIPI CSI → RKISP → /dev/video0
                         ↓
                   UYVY 640×480
                         ↓
                    ~30 FPS
```

La captura se configura automáticamente al arrancar y la imagen útil queda limitada al anillo del espejo. La detección de colores todavía se calibra después, desde la PC.

## Antes de comenzar

Comprueba que:

- Ubuntu Server ya arranca correctamente y tienes acceso SSH.
- La Radxa está apagada antes de conectar el cable MIPI CSI.
- El cable y el adaptador son los adecuados para el conector de la Zero 3W.
- Están instalados `v4l2-ctl`, `media-ctl` y `ffmpeg`.

Si falta alguna herramienta:

```bash
sudo apt install -y v4l-utils ffmpeg
```

No conectes ni desconectes el cable CSI con la placa encendida.

## 1. Activar y detectar la cámara

La cámara se conectó al puerto MIPI CSI con la Radxa apagada. Primero se comprobó que el overlay específico estuviera instalado:

```text
/lib/firmware/6.1.0-1025-rockchip/device-tree/rockchip/overlay/radxa-zero3-rpi-camera-v1.3.dtbo
```

Se hizo una copia de seguridad y se configuró en `/etc/default/u-boot`:

```bash
sudo cp /etc/default/u-boot /etc/default/u-boot.backup-camera
sudo nano /etc/default/u-boot
```

```text
U_BOOT_FDT_OVERLAYS="device-tree/rockchip/overlay/radxa-zero3-rpi-camera-v1.3.dtbo"
```

Después:

```bash
sudo u-boot-update
grep -iE "fdtoverlays|camera|5647" /boot/extlinux/extlinux.conf
sudo reboot
```

La comprobación de `extlinux.conf` se hace antes del reinicio. Si el archivo no contiene el overlay, no reinicies: revisa la línea `U_BOOT_FDT_OVERLAYS`.

El kernel confirmó la cadena:

```text
OV5647 → MIPI CSI → CSI2 D-PHY → RKISP → V4L2
```

La comprobación se hizo con:

```bash
sudo dmesg | grep -iE "ov5647|camera|csi|mipi|rkisp|rkcif"
ls -l /dev/video*
ls -l /dev/media*
v4l2-ctl --list-devices
```

Los nodos relevantes fueron `/dev/video0` (`rkisp_mainpath`), `/dev/video1` (`rkisp_selfpath`), `/dev/video8` (estadísticas), `/dev/video9` (parámetros) y `/dev/media0` (media controller). Los números pueden cambiar en otra imagen o kernel; por eso siempre se debe confirmar con `v4l2-ctl --list-devices`.

La prueba se considera correcta si `dmesg` contiene `OmniVision OV5647 camera driver probed` y aparecen `/dev/video0` y `/dev/media0`.

## 2. Elegir el backend y hacer una captura

Se probó `libcamera` con `cam -l`, pero no registró ninguna cámara utilizable en esta instalación. La captura sí funcionó mediante V4L2/RKISP, por lo que el backend elegido para QUPA es:

```text
/dev/video0 → V4L2 → OpenCV C++
```

No se copiaron `picamera2`, `RPi.GPIO` ni instrucciones específicas de Raspberry Pi.

Para capturar un frame UYVY de 640×480:

```bash
v4l2-ctl \
  -d /dev/video0 \
  --set-fmt-video=width=640,height=480,pixelformat=UYVY \
  --stream-mmap=4 \
  --stream-skip=30 \
  --stream-count=1 \
  --stream-to=$HOME/test_camera.uyvy

ffmpeg -loglevel error -y \
  -f rawvideo -pixel_format uyvy422 -video_size 640x480 \
  -i ~/test_camera.uyvy -frames:v 1 -update 1 ~/test_camera.png
```

Desde la PC se copia el resultado con:

```powershell
scp ubuntu@IP_DE_LA_RADXA:~/test_camera.png .
```

La captura mostró correctamente el anillo reflejado del espejo, aunque inicialmente presentaba dominante verde y zonas saturadas. Esto demuestra que el hardware y la captura funcionan; no significa todavía que los colores estén calibrados.

Si `/dev/video0` no existe, no continúes con los comandos de captura: vuelve a la sección anterior y revisa el overlay, `dmesg` y `v4l2-ctl --list-devices`.

## 3. Ajustar exposición, ganancia y balance

Los controles se inspeccionaron con:

```bash
v4l2-ctl -d /dev/video0 --list-ctrls-menus
v4l2-ctl -d /dev/v4l-subdev3 --list-ctrls-menus
```

En este driver, `auto_exposure=0` es modo automático y `auto_exposure=1` es modo manual. La configuración provisional que mejor equilibró detalle, brillo y ruido fue:

```bash
v4l2-ctl -d /dev/video0 \
  --set-ctrl=auto_exposure=1,gain_automatic=0,white_balance_automatic=1,exposure=1000,analogue_gain=80
```

```yaml
camera:
  exposure: 1000
  analogue_gain: 80
  white_balance_automatic: true
```

Se probaron exposiciones de 250, 500, 750, 850, 900, 950 y 1000, y ganancias de 32, 48, 64, 72, 80, 88 y 96. El valor 80 fue el punto de trabajo provisional. No copies esos valores sin revisar la imagen de tu propia cámara: la iluminación y el espejo pueden cambiar el resultado.

## 4. Seleccionar resolución y FPS

Al inicio, aunque la salida era 640×480, el sensor seguía trabajando en 2592×1944 a unos 15 FPS y el RKISP reducía la imagen. Se configuró el pipeline para que el sensor trabajara directamente en 640×480 con formato Bayer `SGBRG10_1X10`.

La medición real se hizo con streaming, no solo con el FPS reportado por el subdispositivo:

```bash
v4l2-ctl \
  -d /dev/video0 \
  --set-fmt-video=width=640,height=480,pixelformat=UYVY \
  --stream-mmap=4 --stream-count=300 --stream-to=/dev/null
```

El modo nativo de 640×480 entregó aproximadamente 60 FPS reales. Para la detección de blobs se eligió una meta cercana a 30 FPS, con más tiempo de exposición y menor carga. Se aumentó el blanking vertical:

```bash
v4l2-ctl -d /dev/video0 --set-ctrl=vertical_blanking=524
```

La configuración objetivo quedó:

```text
640×480 · ~30 FPS · exposure=1000 · analogue_gain=80 · vertical_blanking=524
```

El intento de usar `--set-subdev-fps` no fue compatible con este driver, por lo que se usó `vertical_blanking` y se verificó la frecuencia con una captura real. En otra versión del kernel este control puede comportarse de otra forma; siempre mide el FPS real.

## 5. Aplicar la configuración automáticamente

Se creó `/usr/local/sbin/qupa-camera-init.sh` para esperar `/dev/media0`, localizar `rkisp_mainpath`, configurar el pipeline 640×480, seleccionar UYVY y aplicar blanking, exposición y ganancia. Se probó manualmente con:

```bash
sudo chmod +x /usr/local/sbin/qupa-camera-init.sh
sudo bash -x /usr/local/sbin/qupa-camera-init.sh
```

Después se preparó el servicio `oneshot` `/etc/systemd/system/qupa-camera.service`:

```bash
sudo systemctl daemon-reload
sudo systemctl enable qupa-camera.service
sudo systemctl restart qupa-camera.service
systemctl status qupa-camera.service
```

El estado esperado es `active (exited)`. Es correcto: el servicio configura la cámara y termina. El primer intento configuraba directamente un pad no aceptado del D-PHY y fallaba con `Invalid argument`; la versión corregida evita esa operación.

Para diagnosticar un fallo:

```bash
journalctl -u qupa-camera.service -b --no-pager
sudo bash -x /usr/local/sbin/qupa-camera-init.sh
```

## 6. Calibrar la geometría del espejo

La geometría se calibró en la PC con Python. Los valores obtenidos fueron:

```yaml
camera:
  width: 640
  height: 480

mirror:
  center_x: 307.01
  center_y: 238.47
  inner_radius: 59.52
  outer_radius: 133.35
```

No se asumió que el centro óptico fuera `(320,240)`: quedó desplazado aproximadamente 13 píxeles hacia la izquierda. Después se marcaron con polígonos los dos soportes horizontales y se generaron:

```text
final_mirror_mask.png
final_mirror_roi.png
```

La ROI final conserva el anillo útil, elimina el centro y descarta los soportes para reducir reflejos y falsos positivos. Estos valores son propios de ese montaje: si se mueve el espejo o la cámara, se repite esta calibración antes de calibrar colores.

## 7. Siguiente etapa: colores y blobs

La calibración de colores se hará en la PC con Python, usando marcadores azules y verdes a diferentes ángulos y distancias. Los rangos HSV y la geometría se guardarán en YAML y luego se copiarán a la Radxa. La ejecución continua quedará en C++/OpenCV:

```text
V4L2 → cv::Mat → ROI del espejo → HSV → morfología → contornos → blobs → ROS 2
```

Python se mantiene como herramienta de calibración; `qupa_camera` y `qupa_vision` serán componentes de producción en C++.
