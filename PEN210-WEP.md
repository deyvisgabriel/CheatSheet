### Iniciar modo monitor
    sudo airmon-ng start wlan0
El comando se descompone de la siguiente manera:

**sudo**: ejecuta el comando que sigue con privilegios de superusuario (root), necesarios para realizar cambios en la configuración de los dispositivos de red.

**airmon-ng**: es una herramienta incluida en la suite Aircrack-ng, un conjunto de herramientas para auditoría de seguridad inalámbrica.

**start**: es la acción que le indicas a airmon-ng que realice, en este caso, iniciar el modo monitor.

**wlan0**: es el nombre de la interfaz de red inalámbrica en la que se activará el modo monitor. Este nombre puede variar dependiendo de la configuración del sistema y de los dispositivos de red disponibles.

### Iniciar captura y monitoreo de todo
    sudo airodump-ng wlan0mon
### Iniciar captura y monitore


