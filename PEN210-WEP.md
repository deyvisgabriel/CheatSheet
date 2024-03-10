### 1. Iniciar modo monitor
    sudo airmon-ng start wlan0

- **airmon-ng:** es una herramienta incluida en la suite Aircrack-ng, un conjunto de herramientas para auditoría de seguridad inalámbrica.

- **start:** es la acción que le indicas a airmon-ng que realice, en este caso, iniciar el modo monitor.

- **wlan0:** es el nombre de la interfaz de red inalámbrica en la que se activará el modo monitor. Este nombre puede variar dependiendo de la configuración del sistema y de los dispositivos de red disponibles.

### 2. Iniciar captura y monitoreo de todo
    sudo airodump-ng wlan0mon

- **airodump-ng:** Es la herramienta dentro de Aircrack-ng dedicada a la captura y visualización en tiempo real de paquetes de redes Wi-Fi.

- **wlan0mon:** Es el nombre de la interfaz de red inalámbrica que se ha colocado previamente en modo monitor.

### 3. Iniciar captura y monitoreo de una red específica

    sudo airodump-ng -c 1 --bssid F0:9F:C2:AA:19:29 -w wifi-old wlan0mon

- **airodump-ng:** Es la herramienta dentro de Aircrack-ng que permite capturar paquetes de redes Wi-Fi y monitorizar el tráfico inalámbrico en tiempo real.

- **-c 1:** Este parámetro le indica a airodump-ng que filtre y escuche solamente en el canal 1.

- **--bssid F0:9F:C2:AA:19:29:** Este parámetro filtra la captura por el BSSID, que es la dirección MAC de la red Wi-Fi, en este caso, "F0:9F:C2:AA:19:29".

- **-w wifi-corp:** Esta opción indica a airodump-ng que guarde los datos capturados en archivos con el prefijo "wifi-corp".

- **wlan0mon:** Es el nombre de la interfaz de red inalámbrica en modo monitor.

### 4. Realizar autenticación falsa (fake authentication)

    sudo aireplay-ng -1 3600 -q 10 -a F0:9F:C2:AA:19:29 wlan0mon

- **aireplay-ng:** Es la herramienta de la suite Aircrack-ng que permite generar tráfico de red.

- **-1:** Este es el modo de autenticación falsa en aireplay-ng. La autenticación falsa se utiliza para asociarse con un punto de acceso, intentando hacer creer al AP que el dispositivo que realiza el ataque es un cliente legítimo.

- **3600:** Este número indica la duración del ataque en segundos. En este caso, 3600 segundos equivalen a una hora.

- **-q 10:** Este parámetro especifica el intervalo en segundos entre los envíos de paquetes de mantenimiento de la autenticación. En este caso, 10 segundos.

- **-a F0:9F:C2:AA:19:29:** Este es el BSSID (identificador único del punto de acceso) al cual se intenta autenticar.

- **wlan0mon:** El nombre de la interfaz de red que se está utilizando para el ataque.

### 5. Incrementar la cantidad de paquetes IVs (Initialization Vectors) capturados

    sudo aireplay-ng --arpreplay -b F0:9F:C2:AA:19:29 -h BA:49:A9:53:A1:8C wlan0mon

- **aireplay-ng:** Es la herramienta específica de la suite Aircrack-ng utilizada para inyectar tráfico en una red con el fin de generar datos que luego pueden ser utilizados en ataques de desencriptación.

- **--arpreplay:** Este es el modo de ataque específico que se está utilizando. El ataque de replay de ARP se concentra en capturar una solicitud ARP de un cliente legítimo y luego retransmitirla múltiples veces. Esto induce al punto de acceso a responder a cada retransmisión con un nuevo paquete, generando rápidamente una gran cantidad de datos con IVs únicos que pueden ser utilizados para intentar romper la encriptación WEP.

- **-b F0:9F:C2:AA:19:29:** Este es el BSSID del punto de acceso objetivo. El BSSID es el identificador único de la red que se está atacando, y es necesario para dirigir correctamente el tráfico inyectado.

- **-h BA:49:A9:53:A1:8C:** Esta es la dirección MAC del atacante (en este contexto, se finge ser la dirección MAC de un cliente legítimo en la red). Este parámetro es necesario para que los paquetes inyectados parezcan provenir de un cliente legítimo de la red, permitiendo que el ataque pase desapercibido.

- **wlan0mon:** Es el nombre de la interfaz de red en modo monitor. Esta interfaz captura paquetes y, en este contexto, también inyecta tráfico en la red.

### 6. Crackear la contraseña de una red Wi-Fi WEP

    sudo aircrack-ng wifi-old-01.cap

- **aircrack-ng:** Es el componente de la suite Aircrack-ng que se utiliza para el crackeo de claves. Puede trabajar con varios algoritmos de encriptación, incluyendo WEP y WPA/WPA2 PSK (Pre-Shared Key).

- **wifi-old-01.cap:** Es el nombre del archivo de captura que contiene los datos de tráfico de red recolectados previamente.

### 7. Conexión a la red WEP
```
sudo airmon-ng stop wlan0mon
```
```
sudo nano wep.conf
```
```
network={
  ssid="wifi-old"
  key_mgmt=NONE
  wep_key0=$PASSWORD
  wep_tx_keyidx=0
}
```
```
sudo wpa_supplicant -D nl80211 -i wlan0 -c wep.conf
```
```
sudo dhclient wlan0 -v
```
### Desautenticación en redes Wi-Fi

    sudo aireplay-ng -0 0 -a F0:9F:C2:71:22:1A -c 64:32:A8:07:6C:40 wlan0mon

**aireplay-ng:** Es la herramienta de la suite Aircrack-ng diseñada para inyectar paquetes en una red Wi-Fi, lo que permite realizar una variedad de ataques.

**-0:** Especifica el tipo de ataque a realizar, en este caso, un ataque de desautenticación. El número 0 después de -0 indica este tipo de ataque.

**0:** La cantidad de paquetes de desautenticación a enviar. Un valor de 0 indica que aireplay-ng enviará paquetes de desautenticación de manera continua hasta que el proceso sea detenido manualmente por el usuario.

**-a F0:9F:C2:71:22:1A:** La dirección MAC del punto de acceso (AP) objetivo. -a especifica la dirección MAC del AP al que el dispositivo cliente está conectado o intenta conectarse.

**-c 64:32:A8:07:6C:40:** La dirección MAC del dispositivo cliente (estación) objetivo. -c se utiliza para especificar la dirección MAC del cliente que queremos desautenticar de la red. Si este parámetro se omite, el ataque se dirigirá a todos los dispositivos conectados al AP especificado.

**wlan0mon:** El nombre de la interfaz de red inalámbrica que está en modo monitor y será utilizada para realizar el ataque.
