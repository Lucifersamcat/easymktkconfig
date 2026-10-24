# Guía Paso a Paso del Proyecto de Clase: `JANIO-SW` (Cisco 2960-S) + `Mkt-Janio` (MikroTik hAP lite)

Esta guía está diseñada **exactamente para tu equipo físico y tu direccionamiento (`172.22.X.0/24`)**, explicando tanto los comandos por **CLI** como los pasos en **WinBox** (RouterOS v6) y en la **consola de Cisco IOS**.

---

## 1. Arquitectura Física y Solución al Límite de 4 Puertos del hAP lite

**¿Cómo conectar 5 cables RJ45 si el MikroTik hAP lite solo tiene 4 puertos físicos (`ether1` a `ether4`)?**
En la imagen del simulador los routers tienen 5 puertos (`eth1` a `eth5`), pero tu **hAP lite físico solo tiene 4 puertos RJ45 (`ether1`, `ether2`, `ether3`, `ether4`) + WiFi (`wlan1`)**:
- **Caso A (Si solo usan `wlan1` para Internet y NO conectan cable WAN):** Te quedan los 4 puertos RJ45 exactos: `ether2` y `ether3` para el Bonding al Switch, `ether1` para el Vecino 1 y `ether4` para el Vecino 2.
- **Caso B (Si conectas Internet por cable en `ether1` + WiFi `wlan1` como Backup):** Usas `ether1` (WAN Cable), `ether2` y `ether3` (Bonding al Switch) y `ether4` (Vecino 1). **¿Y dónde conectas el cable del Vecino 2?** ¡Lo conectas al **puerto 24 de tu Switch Cisco 2960-S** dentro de una **VLAN dedicada de tránsito (`VLAN 40`)**! Esa `VLAN 40` sube por el mismo Troncal Bonding hacia el MikroTik y funciona exactamente igual que un puerto `ether5` dedicado.

### Tabla Maestra de Direccionamiento de `JANIO-SW`
| Red / Enlace | VLAN ID | Nombre de VLAN | IP del MikroTik (Gateway) | Rango DHCP Server | IP del Switch Cisco |
| :--- | :---: | :--- | :--- | :--- | :--- |
| **Estudiantes** | `10` | `estudiantes` | `172.22.10.1/24` | `172.22.10.10 - 172.22.10.200` | N/A |
| **Docentes** | `20` | `docentes` | `172.22.20.1/24` | `172.22.20.10 - 172.22.20.200` | N/A |
| **Administración** | `30` | `administracion` | `172.22.30.1/24` | `172.22.30.10 - 172.22.30.200` | **`172.22.30.2/24`** *(SVI)* |
| **Enlace Vecino 1** | Nativo (`ether4`) | Directo en `ether4` | `10.0.0.1/30` *(ejemplo)* | Sin DHCP (Enlace `/30`) | N/A |
| **Enlace Vecino 2** | `40` *(o `ether1`)* | `vecino2_wan` | `10.0.0.9/30` *(ejemplo)* | Sin DHCP (Enlace `/30`) | Solo puente L2 (Puerto 24) |

---

## FASE 1: Configurar el Switch Cisco 2960-S (`JANIO-SW`)

Conéctate por cable de consola al Switch Cisco 2960-S usando **MobaXterm** o **PuTTY** (`9600 bps`).

### Paso 1.1: Verificar los nombres exactos de los puertos de tu 2960-S
Antes de empezar, ejecuta en el switch:
```ios
Switch> enable
Switch# show ip interface brief
```
- En un **Catalyst 2960-S**, los 24 puertos suelen llamarse **`GigabitEthernet1/0/1` al `1/0/24`** (se abrevia `gi1/0/1`).
- En un **Catalyst 2960 estándar**, se llaman **`FastEthernet0/1` al `0/24`** (se abrevia `fa0/1`) y los 2 puertos Gigabit finales son `gi0/1` y `gi0/2`.
En el script de abajo usamos `gi1/0/X` (propio del 2960-S); si tu switch muestra `FastEthernet0/X`, simplemente escribe `fa0/X` en lugar de `gi1/0/X`.

### Paso 1.2: Borrar configuración previa (si tiene algo de otra clase)
```ios
Switch# delete flash:vlan.dat
Switch# erase startup-config
Switch# reload
```

### Paso 1.3: Nombre, Contraseñas y SSH v2 en `JANIO-SW`
```ios
Switch> enable
Switch# configure terminal

! 1. Nombre del Switch y desactivar búsqueda DNS
Switch(config)# hostname JANIO-SW
JANIO-SW(config)# no ip domain-lookup

! 2. Contraseñas y cifrado
JANIO-SW(config)# enable secret class
JANIO-SW(config)# service password-encryption
JANIO-SW(config)# banner motd #Acceso exclusivo autorizados - Switch JANIO-SW#

! 3. Configurar Consola
JANIO-SW(config)# line console 0
JANIO-SW(config-line)# password cisco
JANIO-SW(config-line)# login
JANIO-SW(config-line)# logging synchronous
JANIO-SW(config-line)# exit

! 4. Configurar SSH v2 y líneas VTY para acceso remoto
JANIO-SW(config)# ip domain-name cano.local
JANIO-SW(config)# username admin privilege 15 secret cisco123
JANIO-SW(config)# crypto key generate rsa general-keys modulus 1024
JANIO-SW(config)# ip ssh version 2

JANIO-SW(config)# line vty 0 15
JANIO-SW(config-line)# transport input ssh telnet
JANIO-SW(config-line)# login local
JANIO-SW(config-line)# logging synchronous
JANIO-SW(config-line)# exit
```

### Paso 1.4: Crear las VLANs (10, 20, 30 y 40) y la SVI de Administración
```ios
! Crear las VLANs con los nombres exactos pedidos
JANIO-SW(config)# vlan 10
JANIO-SW(config-vlan)# name estudiantes
JANIO-SW(config-vlan)# vlan 20
JANIO-SW(config-vlan)# name docentes
JANIO-SW(config-vlan)# vlan 30
JANIO-SW(config-vlan)# name administracion
JANIO-SW(config-vlan)# vlan 40
JANIO-SW(config-vlan)# name vecino2_wan
JANIO-SW(config-vlan)# exit

! Configurar la IP administrativa (SVI) del Switch en la VLAN 30 (administracion)
JANIO-SW(config)# interface vlan 30
JANIO-SW(config-if)# description SVI de Administracion JANIO-SW
JANIO-SW(config-if)# ip address 172.22.30.2 255.255.255.0
JANIO-SW(config-if)# no shutdown
JANIO-SW(config-if)# exit

! Configurar como Default Gateway la IP del MikroTik en la VLAN 30
JANIO-SW(config)# ip default-gateway 172.22.30.1
```

### Paso 1.5: Configurar el EtherChannel LACP (`Port-channel 1`) + Troncal 802.1Q hacia el MikroTik
Usaremos los **puertos 1 y 2** del Switch (`gi1/0/1 - 2`) para conectar a `ether2` y `ether3` del MikroTik:

```ios
! 1. Agrupar los puertos 1 y 2 en LACP (mode active)
JANIO-SW(config)# interface range gi1/0/1 - 2
JANIO-SW(config-if-range)# description Enlace Bonding LACP hacia MikroTik (ether2 y ether3)
JANIO-SW(config-if-range)# shutdown
JANIO-SW(config-if-range)# channel-group 1 mode active
JANIO-SW(config-if-range)# exit

! 2. Configurar la interfaz lógica Port-channel 1 como Troncal 802.1Q
JANIO-SW(config)# interface port-channel 1
JANIO-SW(config-if)# description Troncal 802.1Q hacia MikroTik
JANIO-SW(config-if)# switchport mode trunk
JANIO-SW(config-if)# switchport trunk allowed vlan 10,20,30,40
JANIO-SW(config-if)# exit

! 3. Encender los puertos físicos del EtherChannel
JANIO-SW(config)# interface range gi1/0/1 - 2
JANIO-SW(config-if-range)# no shutdown
JANIO-SW(config-if-range)# exit
```

### Paso 1.6: Asignar los Puertos de Acceso a las VLANs 10, 20, 30 y 40
- **Puertos `3 al 8`:** `VLAN 10 (estudiantes)`
- **Puertos `9 al 16`:** `VLAN 20 (docentes)`
- **Puertos `17 al 23`:** `VLAN 30 (administracion)`
- **Puerto `24`:** `VLAN 40 (vecino2_wan)`

```ios
! Puertos 3 al 8 -> VLAN 10 (estudiantes)
JANIO-SW(config)# interface range gi1/0/3 - 8
JANIO-SW(config-if-range)# description Puertos VLAN 10 Estudiantes
JANIO-SW(config-if-range)# switchport mode access
JANIO-SW(config-if-range)# switchport access vlan 10
JANIO-SW(config-if-range)# spanning-tree portfast
JANIO-SW(config-if-range)# no shutdown
JANIO-SW(config-if-range)# exit

! Puertos 9 al 16 -> VLAN 20 (docentes)
JANIO-SW(config)# interface range gi1/0/9 - 16
JANIO-SW(config-if-range)# description Puertos VLAN 20 Docentes
JANIO-SW(config-if-range)# switchport mode access
JANIO-SW(config-if-range)# switchport access vlan 20
JANIO-SW(config-if-range)# spanning-tree portfast
JANIO-SW(config-if-range)# no shutdown
JANIO-SW(config-if-range)# exit

! Puertos 17 al 23 -> VLAN 30 (administracion)
JANIO-SW(config)# interface range gi1/0/17 - 23
JANIO-SW(config-if-range)# description Puertos VLAN 30 Administracion
JANIO-SW(config-if-range)# switchport mode access
JANIO-SW(config-if-range)# switchport access vlan 30
JANIO-SW(config-if-range)# spanning-tree portfast
JANIO-SW(config-if-range)# no shutdown
JANIO-SW(config-if-range)# exit

! Puerto 24 -> VLAN 40 (Puente hacia el Router del Vecino 2)
JANIO-SW(config)# interface gi1/0/24
JANIO-SW(config-if)# description Enlace hacia Router Vecino 2
JANIO-SW(config-if)# switchport mode access
JANIO-SW(config-if)# switchport access vlan 40
JANIO-SW(config-if)# spanning-tree portfast
JANIO-SW(config-if)# no cdp enable
JANIO-SW(config-if)# no shutdown
JANIO-SW(config-if)# exit

! Guardar toda la configuración del Switch
JANIO-SW(config)# end
JANIO-SW# copy running-config startup-config
```

---

## FASE 2: Configurar el MikroTik hAP lite (Bonding LACP + VLANs + 3 DHCP Servers)

### Paso 2.1: Dejar el MikroTik en blanco e identificarlo
1. Conecta tu PC temporalmente a `ether4` del MikroTik y entra por **MAC-WinBox**.
2. En WinBox ve a **System -> Reset Configuration**, marca **`No Default Configuration`** y **`Do Not Backup`**, y haz clic en **Reset Configuration** (`/system reset-configuration no-defaults=yes skip-backup=yes`).
3. Vuelve a entrar por **MAC-WinBox** (`admin` sin contraseña), abre **New Terminal** y ejecuta:

```routeros
# Cambiar nombre del router y contraseña de admin
/system identity set name=Mkt-Janio
/user set admin password=mikro123

# Renombrar las interfaces para no confundirte
/interface ethernet
set [ find default-name=ether1 ] name=ether1-WAN
set [ find default-name=ether2 ] name=ether2-SW
set [ find default-name=ether3 ] name=ether3-SW
set [ find default-name=ether4 ] name=ether4-Vecino1
```

### Paso 2.2: Crear el Bonding LACP (`802.3ad`) con `ether2-SW` y `ether3-SW`
```routeros
/interface bonding
add name=bonding1 slaves=ether2-SW,ether3-SW mode=802.3ad lacp-rate=30secs transmit-hash-policy=layer-2-and-3
```

### Paso 2.3: Crear las VLANs (10, 20, 30 y 40) sobre `bonding1`
```routeros
/interface vlan
add name=vlan10-estudiantes vlan-id=10 interface=bonding1
add name=vlan20-docentes vlan-id=20 interface=bonding1
add name=vlan30-administracion vlan-id=30 interface=bonding1
add name=vlan40-vecino2 vlan-id=40 interface=bonding1
```

### Paso 2.4: Asignar las Direcciones IP Gateway (`.1/24`) a las 3 VLANs
```routeros
/ip address
add address=172.22.10.1/24 interface=vlan10-estudiantes comment="Gateway VLAN 10 Estudiantes"
add address=172.22.20.1/24 interface=vlan20-docentes comment="Gateway VLAN 20 Docentes"
add address=172.22.30.1/24 interface=vlan30-administracion comment="Gateway VLAN 30 Administracion"
```

### Paso 2.5: Crear los 3 Servidores DHCP (uno para cada VLAN)
```routeros
# 1. Crear los 3 Pools de direcciones IP
/ip pool
add name=pool-vlan10 ranges=172.22.10.10-172.22.10.200
add name=pool-vlan20 ranges=172.22.20.10-172.22.20.200
add name=pool-vlan30 ranges=172.22.30.10-172.22.30.200

# 2. Activar el DHCP Server en cada interfaz VLAN
/ip dhcp-server
add name=dhcp-estudiantes interface=vlan10-estudiantes address-pool=pool-vlan10 lease-time=1h disabled=no
add name=dhcp-docentes interface=vlan20-docentes address-pool=pool-vlan20 lease-time=1h disabled=no
add name=dhcp-admin interface=vlan30-administracion address-pool=pool-vlan30 lease-time=1h disabled=no

# 3. Definir Gateway y DNS que se entregarán a las PCs de cada VLAN
/ip dhcp-server network
add address=172.22.10.0/24 gateway=172.22.10.1 dns-server=172.22.10.1,8.8.8.8
add address=172.22.20.0/24 gateway=172.22.20.1 dns-server=172.22.20.1,8.8.8.8
add address=172.22.30.0/24 gateway=172.22.30.1 dns-server=172.22.30.1,8.8.8.8
```

---

## FASE 3: Doble Conexión a Internet con Failover (Cable Principal + WiFi Backup)

### Paso 3.1: Configurar `ether1-WAN` (Internet por Cable - Distancia 1)
```routeros
/ip dhcp-client
add interface=ether1-WAN add-default-route=yes default-route-distance=1 use-peer-dns=yes disabled=no
```

### Paso 3.2: Configurar `wlan1` (Internet por WiFi como Backup - Distancia 2)
```routeros
/interface wireless security-profiles
add name=perfil-wifi-clase mode=dynamic-keys authentication-types=wpa-psk,wpa2-psk \
    wpa-pre-shared-key="CLAVE_WIFI" wpa2-pre-shared-key="CLAVE_WIFI"

/interface wireless
set [ find default-name=wlan1 ] mode=station band=2ghz-b/g/n ssid="NOMBRE_WIFI" \
    security-profile=perfil-wifi-clase disabled=no

# Cliente DHCP en wlan1 con Distancia 2 (Backup)
/ip dhcp-client
add interface=wlan1 add-default-route=yes default-route-distance=2 use-peer-dns=yes disabled=no
```

### Paso 3.3: Configurar DNS y NAT (`Masquerade`) para AMBAS salidas WAN
```routeros
/ip dns set servers=8.8.8.8,8.8.4.4 allow-remote-requests=yes

/ip firewall nat
add chain=srcnat out-interface=ether1-WAN action=masquerade comment="NAT Salida Principal Cable"
add chain=srcnat out-interface=wlan1 action=masquerade comment="NAT Salida Backup WiFi"
```

---

## FASE 4: Conexión y Enrutamiento con tus 2 Vecinos

### Paso 4.1: Asignar las IPs `/30` de los enlaces con tus vecinos
```routeros
/ip address
add address=10.0.0.1/30 interface=ether4-Vecino1 comment="Enlace /30 hacia Vecino 1"
add address=10.0.0.9/30 interface=vlan40-vecino2 comment="Enlace /30 hacia Vecino 2"
```

### Opción A: Enrutamiento Estático con Respaldo en Triángulo (`/ip route`)
```routeros
# --- RUTAS HACIA LAS 3 VLANs DEL VECINO 1 (Ej. 172.21.X.0/24 por 10.0.0.2) ---
/ip route
add dst-address=172.21.10.0/24 gateway=10.0.0.2 distance=1 check-gateway=ping comment="Vecino 1 VLAN 10"
add dst-address=172.21.20.0/24 gateway=10.0.0.2 distance=1 check-gateway=ping comment="Vecino 1 VLAN 20"
add dst-address=172.21.30.0/24 gateway=10.0.0.2 distance=1 check-gateway=ping comment="Vecino 1 VLAN 30"

# Respaldo hacia Vecino 1 pasando por Vecino 2 (Distancia 2)
add dst-address=172.21.10.0/24 gateway=10.0.0.10 distance=2 comment="Backup Vecino 1 VLAN 10 via Vecino 2"
add dst-address=172.21.20.0/24 gateway=10.0.0.10 distance=2 comment="Backup Vecino 1 VLAN 20 via Vecino 2"
add dst-address=172.21.30.0/24 gateway=10.0.0.10 distance=2 comment="Backup Vecino 1 VLAN 30 via Vecino 2"

# --- RUTAS HACIA LAS 3 VLANs DEL VECINO 2 (Ej. 172.23.X.0/24 por 10.0.0.10) ---
/ip route
add dst-address=172.23.10.0/24 gateway=10.0.0.10 distance=1 check-gateway=ping comment="Vecino 2 VLAN 10"
add dst-address=172.23.20.0/24 gateway=10.0.0.10 distance=1 check-gateway=ping comment="Vecino 2 VLAN 20"
add dst-address=172.23.30.0/24 gateway=10.0.0.10 distance=1 check-gateway=ping comment="Vecino 2 VLAN 30"

# Respaldo hacia Vecino 2 pasando por Vecino 1 (Distancia 2)
add dst-address=172.23.10.0/24 gateway=10.0.0.2 distance=2 comment="Backup Vecino 2 VLAN 10 via Vecino 1"
add dst-address=172.23.20.0/24 gateway=10.0.0.2 distance=2 comment="Backup Vecino 2 VLAN 20 via Vecino 1"
add dst-address=172.23.30.0/24 gateway=10.0.0.2 distance=2 comment="Backup Vecino 2 VLAN 30 via Vecino 1"
```

### Opción B: Enrutamiento Dinámico con OSPF (en RouterOS v6)
```routeros
/routing ospf instance
set [ find default=yes ] router-id=10.0.0.1

/routing ospf network
add network=172.22.10.0/24 area=backbone
add network=172.22.20.0/24 area=backbone
add network=172.22.30.0/24 area=backbone
add network=10.0.0.0/30 area=backbone
add network=10.0.0.8/30 area=backbone
```

### Opción C: Si el profesor pide VPN IPsec Site-to-Site hacia un Vecino
```routeros
/ip ipsec profile
add name=IPSEC1 hash-algorithm=sha1 enc-algorithm=aes-256 dh-group=modp1024 lifetime=8h

/ip ipsec peer
add name=Vecino1 address=10.0.0.2 profile=IPSEC1 exchange-mode=main

/ip ipsec identity
add peer=Vecino1 auth-method=pre-shared-key secret="Curso!!123!!$"

/ip ipsec proposal
add name=PRO-IPSEC1 auth-algorithms=sha1 enc-algorithms=aes-256-cbc pfs-group=modp1024

/ip ipsec policy
add peer=Vecino1 tunnel=yes src-address=172.22.0.0/16 dst-address=172.21.0.0/16 \
    action=encrypt level=require ipsec-protocols=esp proposal=PRO-IPSEC1

/ip firewall nat
add chain=srcnat src-address=172.22.0.0/16 dst-address=172.21.0.0/16 action=accept place-before=0
```
