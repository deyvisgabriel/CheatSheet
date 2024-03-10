### Iniciar modo monitor
    sudo airmon-ng start wlan0

**airmon-ng:** es una herramienta incluida en la suite Aircrack-ng, un conjunto de herramientas para auditoría de seguridad inalámbrica.

**start:** es la acción que le indicas a airmon-ng que realice, en este caso, iniciar el modo monitor.

**wlan0:** es el nombre de la interfaz de red inalámbrica en la que se activará el modo monitor. Este nombre puede variar dependiendo de la configuración del sistema y de los dispositivos de red disponibles.

### Iniciar captura y monitoreo de todo
    sudo airodump-ng wlan0mon

**airodump-ng:** Es la herramienta dentro de Aircrack-ng dedicada a la captura y visualización en tiempo real de paquetes de redes Wi-Fi.

**wlan0mon:** Es el nombre de la interfaz de red inalámbrica que se ha colocado previamente en modo monitor.

### Iniciar captura y monitoreo de una red específica

    sudo airodump-ng -c 44 --essid wifi-corp -w wifi-corp wlan0mon

**airodump-ng:** Es la herramienta dentro de Aircrack-ng que permite capturar paquetes de redes Wi-Fi y monitorizar el tráfico inalámbrico en tiempo real.

**-c 44:** Este parámetro le indica a airodump-ng que filtre y escuche solamente en el canal 44.

**--essid wifi-corp:** Este parámetro filtra la captura por el ESSID (Extended Service Set Identifier), que es el nombre visible de la red Wi-Fi, en este caso, "wifi-corp".

**-w wifi-corp:** Esta opción indica a airodump-ng que guarde los datos capturados en archivos con el prefijo "wifi-corp".

**wlan0mon:** Es el nombre de la interfaz de red inalámbrica en modo monitor.
