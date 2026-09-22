# Configuración base de la Radxa

Esta guía resume la configuración realizada sobre Ubuntu Server 24.04 para la Radxa Zero 3W. Los comandos se ejecutan desde una sesión SSH con el usuario `ubuntu`.

## Verificación inicial

```bash
cat /etc/os-release
uname -a
ip -br a
free -h
lsblk
```

La instalación documentada usa Ubuntu 24.04.5 LTS ARM64 y el kernel `6.1.0-1025-rockchip`.

## Identidad y zona horaria

```bash
sudo hostnamectl set-hostname qupa2
sudo timedatectl set-timezone America/Guayaquil
hostname
timedatectl status
```

La salida esperada de `timedatectl` debe indicar que el reloj está sincronizado y que `systemd-timesyncd` está activo.

## Red y SSH

En la imagen Server, `wlan0` estaba detectada por el sistema, pero inicialmente aparecía `DOWN` porque todavía no había una red Wi-Fi definida. La conexión se hizo editando el archivo de Netplan y colocando allí el nombre de la red y su contraseña.

Primero se hizo una copia de seguridad:

```bash
sudo cp /etc/netplan/50-cloud-init.yaml \
  /etc/netplan/50-cloud-init.yaml.bak
```

Después se editó el archivo:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

La estructura utilizada fue esta. Sustituye los valores entre comillas por el SSID y la contraseña de tu propia red; no guardes credenciales reales en este repositorio:

```yaml
network:
  version: 2

  ethernets:
    zz-all-en:
      match:
        name: "en*"
      optional: true
      dhcp4: true

    zz-all-eth:
      match:
        name: "eth*"
      optional: true
      dhcp4: true

  wifis:
    wlan0:
      optional: true
      dhcp4: true
      access-points:
        "NOMBRE_DE_LA_RED":
          password: "CONTRASENA_DE_LA_RED"
```

Es importante respetar la indentación con espacios y no utilizar tabuladores. La sección `wifis` debe estar al mismo nivel que `ethernets`. Si el SSID contiene espacios, debe conservarse entre comillas.

Antes de aplicar la configuración se validó que el YAML pudiera generarse:

```bash
sudo netplan generate
```

Si no aparece ningún error, se aplica la configuración:

```bash
sudo netplan apply
```

Después se comprueba que `wlan0` esté activa y que haya recibido una dirección IP:

```bash
ip -br a
```

Una salida válida debe mostrar algo parecido a:

```text
wlan0    UP    10.x.x.x/24
```

La dirección concreta depende del router o del punto de acceso. Tener una IP asignada confirma la asociación con la red local, pero todavía hay que comprobar el acceso a Internet.

Primero se prueba conectividad por dirección IP, sin depender de DNS:

```bash
ping -c 3 8.8.8.8
```

Luego se prueba resolución de nombres y salida a Internet:

```bash
ping -c 3 google.com
```

Si ambos comandos responden, la Radxa tiene conexión de red e Internet funcionando. Durante esta configuración el comando `ping` inicialmente mostró un error de permisos (`Operation not permitted`); se corrigió reinstalando el paquete y restaurando la capacidad necesaria:

```bash
sudo apt install --reinstall -y iputils-ping
sudo setcap cap_net_raw+ep /usr/bin/ping
getcap /usr/bin/ping
```

La última comprobación debe mostrar `cap_net_raw=ep`.

El archivo contiene la contraseña de la red, por eso se protegió después de editarlo:

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
ls -l /etc/netplan/
```

Finalmente se habilitaron los servicios para continuar administrando la Radxa remotamente:

```bash
sudo systemctl enable --now ssh
sudo systemctl enable --now avahi-daemon
```

Durante las pruebas se utilizó `wlan0` y la conexión se verificó con:

```bash
ping -c 3 google.com
```

## Herramientas base

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
  git curl wget nano vim htop tmux tree \
  build-essential cmake pkg-config gdb clang-format cppcheck ccache \
  python3 python3-pip python3-venv python3-dev \
  openssh-server avahi-daemon i2c-tools usbutils pciutils \
  v4l-utils libcamera-tools picocom libboost-system-dev libudev-dev
```

## ROS 2 y bibliotecas

La Radxa utiliza ROS 2 Jazzy en modo `ros-base`; las herramientas gráficas se mantienen en la PC.

```bash
sudo apt install -y \
  ros-jazzy-ros-base ros-dev-tools \
  ros-jazzy-rmw-cyclonedds-cpp \
  ros-jazzy-tf2-ros ros-jazzy-robot-state-publisher \
  ros-jazzy-xacro ros-jazzy-diagnostic-updater \
  ros-jazzy-cv-bridge ros-jazzy-image-transport \
  ros-jazzy-camera-info-manager libopencv-dev
```

Comprueba la instalación:

```bash
source /opt/ros/jazzy/setup.bash
printenv ROS_DISTRO
which colcon
pkg-config --modversion opencv4
```

## Entorno QUPA

El archivo `~/qupa_env.bash` debe cargar ROS, seleccionar CycloneDDS y añadir el workspace si ya fue compilado:

```bash
#!/usr/bin/env bash
source /opt/ros/jazzy/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export ROS_DOMAIN_ID=0
export ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET

if [ -f "$HOME/qupa_ws/install/setup.bash" ]; then
  source "$HOME/qupa_ws/install/setup.bash"
fi
```

Actívalo en nuevas sesiones:

```bash
chmod +x ~/qupa_env.bash
echo 'source ~/qupa_env.bash' >> ~/.bashrc
source ~/qupa_env.bash
```

## Workspace

```bash
mkdir -p ~/qupa_ws/src
cd ~/qupa_ws
```

Después de añadir paquetes:

```bash
source ~/qupa_env.bash
colcon build --symlink-install
source install/setup.bash
```

## Overlays y cámara

Antes de activar cualquier overlay, guardar una copia:

```bash
sudo cp /etc/default/u-boot /etc/default/u-boot.backup
```

Inspeccionar primero los archivos disponibles:

```bash
find /lib/firmware/$(uname -r)/device-tree/rockchip/overlay \
  -maxdepth 1 -type f -name '*.dtbo' | sort
```

La prueba de la OV5647 se hará cuando la cámara esté físicamente conectada. No se debe asumir que `/dev/video0` es el dispositivo correcto: comprobar `dmesg`, `/dev/media*`, `v4l2-ctl --list-devices`, `media-ctl` y `cam -l`.

## NPU y memoria

El kernel incluye el driver RKNPU y el sistema tiene un overlay NPU, pero el proyecto actual usa OpenCV clásico para HSV, máscaras, morfología, contornos y tracking. Por ello RKNN no es una dependencia de la configuración base.

ZRAM tampoco es requisito funcional. Se añadirá únicamente si las mediciones de memoria durante compilación, ROS y cámara muestran presión de RAM.
