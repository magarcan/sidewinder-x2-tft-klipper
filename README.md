# Revivir Pantalla Original de Artillery Sidewinder X2 con Orange Pi (Klipper & Moonraker)

Este proyecto documenta cómo dar una segunda vida a la pantalla táctil original (clon de la MKS TFT28) de la impresora 3D **Artillery Sidewinder X2** tras migrar su sistema de Marlin a **Klipper** utilizando una placa **Orange Pi Zero 2**.

Al instalar Klipper en la placa base original de la impresora (Artillery Ruby), la pantalla queda inutilizada (pantalla en negro o con mensaje de "No connection"). Mediante este método, aprovechamos el cableado original de la impresora y un adaptador USB-TTL económico para conectar la pantalla directamente a la Orange Pi como un componente de Moonraker, recuperando la interfaz física de control.

---

## ⚠️ Advertencia de Seguridad Importante

*   **Desconexión total:** Antes de realizar cualquier conexión física, asegúrate de que la impresora está completamente apagada y desenchufada de la corriente eléctrica.
*   **Aislamiento de fuentes:** La pantalla debe recibir alimentación eléctrica de una única fuente. Al conectarla al puerto USB de la Orange Pi a través del adaptador, se alimentará de esta. **Bajo ningún concepto** dejes la pantalla conectada a la toma de corriente original de la placa base mientras la alimentas por USB, ya que esto dañará la electrónica.

---

## BOM (Lista de Materiales)

Para llevar a cabo este proyecto necesitarás:

1.  **Impresora 3D:** Artillery Sidewinder X2 (con placa Artillery Ruby v1.2 y pantalla táctil original).
2.  **SBC:** Orange Pi Zero 2 con Klipper y Moonraker ya instalados y operativos.
3.  **Conversor USB a TTL:** Un adaptador basado en el chip **CP2102** (o en su defecto, CH340 / PL2303).
4.  **Cables puente (Jumpers):** 4 cables Dupont (usualmente de tipo Macho-Hembra o Macho-Macho, dependiendo del método de empalme con el conector original).
5.  **Tarjeta SD:** Una tarjeta de memoria SD (o MicroSD con adaptador) de **8 GB o menor capacidad** formateada en **FAT32** (para flashear la pantalla).

---

## 🔌 Conexión Física (El método limpio y seguro)

En lugar de desmontar la pantalla o modificar el delicado cable plano flexible (FFC) interno, aprovecharemos el conector hembra de 4 pines del cable original que se conecta al puerto `UART` de la placa base Artillery Ruby (ubicado en la parte inferior central de la placa).

Dado que el cableado de fábrica ya realiza el cruce de datos interno, la conexión entre el cable desenchufado y el conversor CP2102 se realiza de forma **directa (1 a 1)** siguiendo la serigrafía impresa en la placa base:

| Pin impreso en placa base (Ruby) | Cable original | Pin en el Conversor CP2102 |
| :--- | :--- | :--- |
| **`RX`** | Señal TX de la pantalla | **`RX`** |
| **`TX`** | Señal RX de la pantalla | **`TX`** |
| **`GND`** | Masa común | **`GND`** |
| **`+5V`** | Alimentación de la pantalla | **`5V`** |

1.  Desconecta el enchufe blanco del puerto `UART` de la placa Artillery Ruby.
2.  Empalma cada uno de los 4 hilos de este conector a tu adaptador CP2102 siguiendo la tabla anterior.
3.  Conecta el adaptador CP2102 a un puerto USB libre de tu Orange Pi Zero 2.

---

## 💾 Fase 1: Flasheo del Firmware de la Pantalla

El firmware original de Artillery espera comandos de Marlin. Debemos actualizar la pantalla con un firmware compatible con el adaptador de Moonraker. Utilizaremos el firmware de código abierto para TFT28 adaptado para Klipper.

1.  Descarga los archivos del repositorio de firmwares compatibles con Klipper (por ejemplo, [kabroxiko/Artillery-TFTFirmware](https://github.com/kabroxiko/Artillery-TFTFirmware)).
2.  Formatea tu tarjeta SD en formato **FAT32**.
3.  Busca el archivo binario del firmware para la TFT28 (ej. `MKS_TFT28_V4.0.27.x.bin`) y cópialo en la raíz de la tarjeta SD.
4.  **Imprescindible:** Renombra el archivo en tu SD exactamente a: **`MKSTFT28.bin`** (para que el cargador de la pantalla lo reconozca).
5.  Copia la carpeta de recursos visuales (carpeta **`TFT28`**) y el archivo de configuración **`config.ini`** a la raíz de la SD.
6.  Abre el archivo `config.ini` con un editor de texto en tu ordenador y asegúrate de que el parámetro de velocidad de comunicación esté configurado en `115200` baudios (`baudrate: 115200`). Guarda los cambios.
7.  Con el sistema apagado, inserta la tarjeta SD en la ranura correspondiente de la pantalla.
8.  Enciende la Orange Pi Zero 2. El CP2102 alimentará la pantalla, la cual iniciará de manera automática el proceso de flasheo. Espera a que la interfaz gráfica se cargue por completo antes de apagar el sistema y retirar la SD.

---

## ⚙️ Fase 2: Configuración del Software en Orange Pi Zero 2

Para la comunicación, integraremos el componente nativo de Moonraker `serial-tft-adapter`.

### 1. Instalación del componente
Accede a tu Orange Pi por SSH y ejecuta los siguientes comandos:

```bash
# Navegar a la carpeta de usuario
cd ~

# Clonar el repositorio del adaptador
git clone https://github.com/moonraker-components/serial-tft-adapter.git

# Crear un enlace simbólico en el directorio de Moonraker
ln -s ~/serial-tft-adapter/components/tftadapter.py ~/moonraker/moonraker/components/tftadapter.py
```

### 2. Identificar el puerto USB
Con el adaptador CP2102 conectado, busca su identificador único escribiendo en la terminal:

```bash
ls /dev/serial/by-id/*
```

Aparecerá una ruta larga similar a la siguiente. Cópiala por completo:
`/dev/serial/by-id/usb-Silicon_Labs_CP2102_USB_to_UART_Bridge_Controller_0001-if00-port0`

### 3. Modificar `moonraker.conf`
Abre la interfaz web de tu impresora (Mainsail o Fluidd), dirígete a la sección de configuración y abre tu archivo **`moonraker.conf`**. Añade al final del archivo las siguientes líneas:

```ini
[tftadapter]
serial: /dev/serial/by-id/usb-Silicon_Labs_CP2102_USB_to_UART_Bridge_Controller_0001-if00-port0
baud: 115200

[update_manager serial-tft-adapter]
type: git_repo
primary_branch: main
path: ~/serial-tft-adapter
origin: https://github.com/moonraker-components/serial-tft-adapter.git
managed_services: moonraker
```
*(Recuerda reemplazar la ruta del parámetro `serial:` por el identificador exacto de tu CP2102 obtenido en el paso anterior).*

### 4. Reiniciar Moonraker
Guarda el archivo de configuración y reinicia el servicio de Moonraker desde la interfaz web o mediante la consola de SSH:

```bash
sudo systemctl restart moonraker
```

Una vez que el servicio de Moonraker vuelva a iniciar, establecerá comunicación con la pantalla táctil mediante el puerto serie virtual y el panel físico mostrará de inmediato los datos de Klipper.

---

## 🔗 Referencias y Agradecimientos

*   [serial-tft-adapter por moonraker-components](https://github.com/moonraker-components/serial-tft-adapter) - Componente de Moonraker que hace posible esta integración.
*   [Artillery-TFTFirmware por kabroxiko](https://github.com/kabroxiko/Artillery-TFTFirmware) - Firmware de código abierto adaptado para las pantallas MKS TFT28/35 de Artillery.
```
