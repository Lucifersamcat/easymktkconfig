# Guía Maestra y Exhaustiva: Configuración de Switches Cisco y Routers MikroTik

Esta guía ha sido diseñada específicamente a partir de **todos los módulos, presentaciones, lecturas y laboratorios prácticos** de tu carpeta del curso **Redes Avanzadas (Canó Academy - Prof. Tomás Rodríguez)**. Cubre desde cero hasta nivel avanzado tanto **Conmutación en Cisco IOS** (Módulos 1 y 2) como **Enrutamiento, VPN IPsec y Seguridad en MikroTik RouterOS** (Módulos 3, 4 y 5), incluyendo las plantillas exactas de comandos para resolver todos tus laboratorios.

---

# PARTE I: SWITCHES CISCO (MÓDULOS 1 Y 2)

## 1. Conceptos Fundamentales y Modos del Switch

Un switch de Capa 2 toma decisiones basadas en la **dirección MAC de destino** utilizando circuitos **ASIC** y su tabla **CAM (tabla de direcciones MAC)**, cuyas entradas dinámicas caducan por defecto a los **300 segundos**.
- **Tres funciones principales:**
  1. **Aprendizaje de MAC:** Lee la **MAC de origen** de cada trama entrante y registra el puerto y VLAN en la tabla MAC.
  2. **Filtrado y Reenvío:** Busca la **MAC de destino** en la tabla MAC. Si está, la envía solo por ese puerto; si no está (*unicast desconocido*), **inunda (floods)** la trama por todos los puertos de la misma VLAN excepto por el que entró.
  3. **Prevención de bucles:** Mediante **STP (Spanning Tree Protocol)** bloquea puertos redundantes para evitar tormentas de broadcast e inestabilidad de la tabla MAC.
- **Métodos de conmutación:**
  - **Almacenamiento y reenvío (*Store-and-Forward*):** Recibe la trama completa, calcula el CRC y verifica el campo **FCS**. Descarta tramas con errores (usado en switches Cisco Catalyst como el 2960).
  - **Corte directo (*Cut-Through*):** Lee solo los primeros 6 bytes (MAC de destino) y reenvía inmediatamente (menor latencia, pero no detecta tramas dañadas).
  - **Libre de fragmentos (*Fragment-Free*):** Lee los primeros **64 bytes** (tamaño mínimo Ethernet) para filtrar colisiones (*runts*) antes de reenviar.

### Modos de CLI en Cisco IOS
| Modo | Prompt | Cómo entrar | Cómo salir |
| :--- | :--- | :--- | :--- |
| **EXEC Usuario** | `Switch>` | Conexión inicial por consola | `logout` o `exit` |
| **EXEC Privilegiado** | `Switch#` | `enable` | `disable` o `exit` |
| **Configuración Global** | `Switch(config)#` | `configure terminal` (`conf t`) | `end` o `exit` o `Ctrl+Z` |
| **Configuración de Interfaz** | `Switch(config-if)#` | `interface f0/1` (o `int range f0/1-24`) | `exit` (un nivel) o `end` (a `Switch#`) |
| **Configuración de Línea** | `Switch(config-line)#` | `line console 0` / `line vty 0 15` | `exit` o `end` |

---

## 2. Configuración Básica del Switch (Lab 1 - Módulo 1)

### 2.1. Inicializar / Borrar un Switch de Fábrica (Apéndice A)
En los switches Cisco las VLANs no se guardan en `startup-config`, sino en el archivo **`vlan.dat`** en la memoria Flash. Para dejar un switch completamente limpio:

```ios
Switch> enable
Switch# delete flash:vlan.dat
Delete filename [vlan.dat]? [Enter]
Delete flash:/vlan.dat? [confirm] [Enter]

Switch# erase startup-config
Erasing the nvram filesystem will remove all configuration files! Continue? [confirm] [Enter]

Switch# reload
System configuration has been modified. Save? [yes/no]: no
Proceed with reload? [confirm] [Enter]
```

### 2.2. Habilitar soporte IPv4 e IPv6 (Plantilla SDM en Catalyst 2960)
Por defecto, el Catalyst 2960 usa la plantilla `default` que no soporta IPv6. Para habilitar IPv6 en la SVI:

```ios
S1# show sdm prefer
S1# configure terminal
S1(config)# sdm prefer dual-ipv4-and-ipv6 default
S1(config)# end
S1# reload
```

### 2.3. Plantilla Completa de Ajustes Básicos, Seguridad de Acceso y SVI
Esta secuencia cubre **todos los pasos estándar** que te piden en la Parte 1 de todos los laboratorios de Cisco:

```ios
Switch> enable
Switch# configure terminal

! 1. Nombre del dispositivo y desactivar búsqueda DNS
Switch(config)# hostname S1
S1(config)# no ip domain-lookup

! 2. Contraseña cifrada de modo EXEC privilegiado (class)
S1(config)# enable secret class

! 3. Asegurar el puerto de Consola
S1(config)# line console 0
S1(config-line)# password cisco
S1(config-line)# login
S1(config-line)# logging synchronous
S1(config-line)# exec-timeout 5 0
S1(config-line)# exit

! 4. Asegurar las líneas de terminal virtual (VTY 0 a 15) para Telnet/SSH
S1(config)# line vty 0 15
S1(config-line)# password cisco
S1(config-line)# login
S1(config-line)# exit

! 5. Cifrar todas las contraseñas en texto plano en el running-config
S1(config)# service password-encryption

! 6. Mensaje de advertencia (Banner MOTD)
S1(config)# banner motd #Acceso restringido. Solo personal autorizado.#

! 7. Configurar la VLAN de Administración (SVI) y Gateway Predeterminado
! Por seguridad nunca se deja la administración en la VLAN 1 predeterminada
S1(config)# vlan 99
S1(config-vlan)# name Management
S1(config-vlan)# exit

S1(config)# interface vlan 99
S1(config-if)# description Interfaz SVI de Administracion
S1(config-if)# ip address 192.168.1.2 255.255.255.0
S1(config-if)# ipv6 address 2001:db8:acad::2/64
S1(config-if)# ipv6 address fe80::2 link-local
S1(config-if)# no shutdown
S1(config-if)# exit

! Gateway predeterminado para poder administrar el switch desde otras subredes
S1(config)# ip default-gateway 192.168.1.1

! 8. Ajustar el reloj (en modo privilegiado) y guardar configuración
S1(config)# end
S1# clock set 09:30:00 23 Oct 2026
S1# copy running-config startup-config
```

### 2.4. Configurar SSH para Administración Remota Segura (Packet Tracer 1.3.6)
Telnet envía todo en texto plano. Para reemplazar Telnet por **SSH v2**:

```ios
S1(config)# hostname S1
S1(config)# ip domain-name canoacademy.com
S1(config)# crypto key generate rsa general-keys modulus 1024
S1(config)# username admin privilege 15 secret admin123
S1(config)# ip ssh version 2

S1(config)# line vty 0 15
S1(config-line)# transport input ssh
S1(config-line)# login local
S1(config-line)# exit
```

### 2.5. Administración de Puertos Físicos y Tabla MAC
```ios
! Configurar velocidad, dúplex y Auto-MDIX en una interfaz
S1(config)# interface fastEthernet 0/6
S1(config-if)# description Conectado a PC-A
S1(config-if)# duplex full
S1(config-if)# speed 100
S1(config-if)# mdix auto
S1(config-if)# no shutdown
S1(config-if)# exit

! Configurar una dirección MAC estática (Parte 4 del Lab 1)
S1(config)# mac address-table static 0050.56BE.6C89 vlan 99 interface fastEthernet 0/6
! Para eliminarla:
S1(config)# no mac address-table static 0050.56BE.6C89 vlan 99 interface fastEthernet 0/6
```

**Comandos de verificación del Lab 1:**
- `show running-config` / `show startup-config`
- `show version` (muestra versión de IOS, nombre de imagen `.bin` y Base ethernet MAC Address)
- `show flash:` (muestra archivos en memoria flash, incluyendo la imagen `.bin` y `vlan.dat`)
- `show ip interface brief` / `show interfaces vlan 99`
- `show mac address-table` / `show mac address-table dynamic` / `clear mac address-table dynamic`

---

## 3. VLANs, Enlaces Troncales 802.1Q y DTP (Labs 2 y 3 - Módulo 1)

### 3.1. Rangos de VLANs en Cisco Catalyst
- **Rango Normal (`1 - 1005`):** Se guardan en `flash:vlan.dat`. La `VLAN 1` (predeterminada) y las `1002 - 1005` (FDDI/Token Ring) vienen creadas por defecto y **no se pueden eliminar**.
- **Rango Extendido (`1006 - 4094`):** Se guardan en `running-config` y requieren que el switch esté en modo `vtp transparent`.

### 3.2. Creación de VLANs y Asignación de Puertos de Acceso (Lab 2 y Lab 3)
```ios
! 1. Crear las VLANs en la base de datos
S1(config)# vlan 10
S1(config-vlan)# name Administracion
S1(config-vlan)# vlan 20
S1(config-vlan)# name Ventas
S1(config-vlan)# vlan 30
S1(config-vlan)# name Operaciones
S1(config-vlan)# vlan 999
S1(config-vlan)# name Parking_Lot
S1(config-vlan)# vlan 1000
S1(config-vlan)# name Native
S1(config-vlan)# exit

! 2. Asignar un puerto de usuario a una VLAN de datos (y opcionalmente VLAN de voz)
S1(config)# interface f0/6
S1(config-if)# switchport mode access
S1(config-if)# switchport access vlan 20
! Si hubiera un teléfono IP Cisco conectado al mismo puerto (VLAN de voz 150):
! S1(config-if)# mls qos trust cos
! S1(config-if)# switchport voice vlan 150
S1(config-if)# exit

! 3. Asegurar y apagar TODOS los puertos no utilizados (Parking_Lot)
S1(config)# interface range f0/2-5, f0/7-24, g0/1-2
S1(config-if-range)# switchport mode access
S1(config-if-range)# switchport access vlan 999
S1(config-if-range)# shutdown
S1(config-if-range)# exit
```

### 3.3. Protocolo DTP (Dynamic Trunking Protocol) y Troncales 802.1Q
**Mejor práctica en producción y en los Labs 2, 3, 4, 5 y 6:** Configurar el enlace troncal manualmente (`switchport mode trunk`), cambiar la VLAN nativa a una distinta de la 1 (ej. `1000` o `333`), restringir las VLANs permitidas y **desactivar DTP (`switchport nonegotiate`)**:

```ios
S1(config)# interface f0/1
S1(config-if)# description Enlace Troncal hacia S2
! Nota: En switches multicapa (ej. 3560/3650 o vIOS-L2 en PNETLab) debes definir primero la encapsulación:
! S1(config-if)# switchport trunk encapsulation dot1q
S1(config-if)# switchport mode trunk
S1(config-if)# switchport trunk native vlan 1000
S1(config-if)# switchport trunk allowed vlan 10,20,30,1000
S1(config-if)# switchport nonegotiate
S1(config-if)# exit
```

**Comandos de verificación de VLANs y Troncales:**
- `show vlan brief` (lista las VLANs y sus puertos de acceso; los puertos troncales no aparecen aquí).
- `show interfaces trunk` (muestra qué puertos son troncales, encapsulación `802.1q`, `Native vlan` y `Vlans allowed`).
- `show interfaces f0/1 switchport` (muestra modo administrativo, modo operativo y si `Negotiation of Trunking` está `On` u `Off`).
- `show dtp interface f0/1`

---

## 4. Enrutamiento Inter-VLAN (Lab 4 - Módulo 1)

### 4.1. Método 1: Router-on-a-Stick (Solución completa del Lab 4)
Un único enlace físico entre el Switch (`F0/5`) y el Router (`G0/0/1`) configurado como troncal 802.1Q.

**1) En el Switch `S1` (configurar el troncal hacia el Router `R1` en `F0/5`):**
```ios
S1(config)# interface f0/5
S1(config-if)# description Troncal hacia R1 G0/0/1
S1(config-if)# switchport mode trunk
S1(config-if)# switchport trunk native vlan 1000
S1(config-if)# switchport trunk allowed vlan 10,20,30,1000
S1(config-if)# switchport nonegotiate
S1(config-if)# no shutdown
S1(config-if)# exit
```

**2) En el Router `R1` (crear subinterfaces 802.1Q y encender la interfaz física):**
```ios
R1(config)# interface g0/0/1.10
R1(config-subif)# description Gateway para VLAN 10 Management
R1(config-subif)# encapsulation dot1Q 10
R1(config-subif)# ip address 192.168.10.1 255.255.255.0
R1(config-subif)# exit

R1(config)# interface g0/0/1.20
R1(config-subif)# description Gateway para VLAN 20 Sales
R1(config-subif)# encapsulation dot1Q 20
R1(config-subif)# ip address 192.168.20.1 255.255.255.0
R1(config-subif)# exit

R1(config)# interface g0/0/1.30
R1(config-subif)# description Gateway para VLAN 30 Operations
R1(config-subif)# encapsulation dot1Q 30
R1(config-subif)# ip address 192.168.30.1 255.255.255.0
R1(config-subif)# exit

! Subinterfaz para la VLAN Nativa (NO lleva dirección IP según el Lab 4)
R1(config)# interface g0/0/1.1000
R1(config-subif)# description VLAN Nativa 1000
R1(config-subif)# encapsulation dot1Q 1000 native
R1(config-subif)# exit

! ¡MUY IMPORTANTE! Encender la interfaz física principal
R1(config)# interface g0/0/1
R1(config-if)# no shutdown
R1(config-if)# exit
```

### 4.2. Método 2: Switch de Capa 3 / Multicapa (Packet Tracer 4.3.8)
En un Switch de Capa 3 (como el Catalyst 3560 o 3650), el enrutamiento se hace por hardware internamente mediante **interfaces SVI** y el comando global **`ip routing`**:

```ios
! 1. Habilitar el enrutamiento IPv4 en el switch multicapa
MLS(config)# ip routing

! 2. Crear las VLANs
MLS(config)# vlan 10
MLS(config-vlan)# name Ventas
MLS(config-vlan)# vlan 20
MLS(config-vlan)# name Operaciones
MLS(config-vlan)# exit

! 3. Crear una SVI por cada VLAN (actúan como Default Gateway de los hosts)
MLS(config)# interface vlan 10
MLS(config-if)# ip address 192.168.10.1 255.255.255.0
MLS(config-if)# no shutdown
MLS(config-if)# interface vlan 20
MLS(config-if)# ip address 192.168.20.1 255.255.255.0
MLS(config-if)# no shutdown
MLS(config-if)# exit

! 4. Si necesitas conectar el Switch Capa 3 a un Router usando un Puerto Enrutado (Capa 3 puro):
MLS(config)# interface g0/1
MLS(config-if)# no switchport
MLS(config-if)# ip address 10.10.10.2 255.255.255.252
MLS(config-if)# no shutdown
MLS(config-if)# exit
```

---

## 5. Conmutación Avanzada: VTP, STP y EtherChannel (Módulo 2)

### 5.1. VTP (VLAN Trunking Protocol)
```ios
S1(config)# vtp mode server
S1(config)# vtp domain CANO
S1(config)# vtp password cisco
S1# show vtp status
```

### 5.2. Spanning Tree Protocol (STP / Rapid PVST+)
**Comandos para configurar Rapid PVST+ y forzar el Root Bridge:**
```ios
S1(config)# spanning-tree mode rapid-pvst
! Hacer que S1 sea el puente raíz principal para la VLAN 10 (baja la prioridad a 24576 + 10 = 24586)
S1(config)# spanning-tree vlan 10 root primary
! O establecer la prioridad manualmente (en múltiplos de 4096):
S1(config)# spanning-tree vlan 10 priority 4096

! Hacer que S2 sea el puente raíz secundario (respaldo, prioridad 28672 + 10)
S2(config)# spanning-tree vlan 10 root secondary

! Verificación
S1# show spanning-tree
S1# show spanning-tree vlan 10
```

### 5.3. EtherChannel (Agregación de Enlaces - Lab 5)
**Configuración paso a paso del Lab 5 (EtherChannel LACP en `F0/1` y `F0/2`):**
```ios
! Aplicar en AMBOS switches (S1 y S2):
S1(config)# interface range f0/1 - 2
S1(config-if-range)# shutdown
S1(config-if-range)# channel-group 1 mode active
S1(config-if-range)# no shutdown
S1(config-if-range)# exit

! Configurar los parámetros troncales directamente en la interfaz lógica Port-channel 1
S1(config)# interface port-channel 1
S1(config-if)# switchport mode trunk
S1(config-if)# switchport trunk native vlan 1000
S1(config-if)# switchport trunk allowed vlan 10,20,1000
S1(config-if)# exit
```

**Verificación de EtherChannel:**
```ios
S1# show etherchannel summary
S1# show interfaces port-channel 1
S1# show interfaces trunk
```

---

## 6. Seguridad del Switch en Capa 2 y SNMP (Lab 6 - Módulo 2)

### 6.1. Port Security (Seguridad de Puertos - Paso 8 del Lab 6)
```ios
! En S1 (Puerto F0/6: máximo 3 MACs, modo restrict, 60 min por inactividad)
S1(config)# interface f0/6
S1(config-if)# switchport mode access
S1(config-if)# switchport access vlan 10
S1(config-if)# switchport port-security
S1(config-if)# switchport port-security maximum 3
S1(config-if)# switchport port-security violation restrict
S1(config-if)# switchport port-security aging time 60
S1(config-if)# switchport port-security aging type inactivity
S1(config-if)# exit

! En S2 (Puerto F0/18: aprendizaje Sticky, máximo 2 MACs, modo protect, 60 min absoluto)
S2(config)# interface f0/18
S2(config-if)# switchport mode access
S2(config-if)# switchport access vlan 10
S2(config-if)# switchport port-security
S2(config-if)# switchport port-security mac-address sticky
S2(config-if)# switchport port-security maximum 2
S2(config-if)# switchport port-security violation protect
S2(config-if)# switchport port-security aging time 60
S2(config-if)# exit
```
- **Verificación:** `show port-security interface f0/6` y `show port-security address`

### 6.2. DHCP Snooping (Inspección DHCP - Paso 9 del Lab 6)
```ios
! 1. Habilitar DHCP Snooping globalmente y en la VLAN 10
S2(config)# ip dhcp snooping
S2(config)# ip dhcp snooping vlan 10

! 2. Configurar como CONFIABLE (trust) el puerto troncal que va hacia el servidor DHCP (R1)
S2(config)# interface f0/1
S2(config-if)# ip dhcp snooping trust
S2(config-if)# exit

! 3. Limitar los puertos de acceso NO confiables a 5 paquetes DHCP por segundo
S2(config)# interface f0/18
S2(config-if)# ip dhcp snooping limit rate 5
S2(config-if)# exit

! En el Router R1 cuando actúa como servidor DHCP:
R1(config)# interface g0/0/1
R1(config-if)# ip dhcp relay information trusted
```
- **Verificación:** `show ip dhcp snooping` y `show ip dhcp snooping binding`

### 6.3. Dynamic ARP Inspection - DAI (Módulo 2 - Sección 11.4)
```ios
S1(config)# ip dhcp snooping
S1(config)# ip dhcp snooping vlan 10
S1(config)# ip arp inspection vlan 10
S1(config)# ip arp inspection validate src-mac dst-mac ip

S1(config)# interface f0/1
S1(config-if)# ip dhcp snooping trust
S1(config-if)# ip arp inspection trust
S1(config-if)# exit
```

### 6.4. PortFast y BPDU Guard (Paso 10 del Lab 6)
```ios
! Por interfaz en los puertos de acceso en uso (F0/5 y F0/6 en S1; F0/18 en S2)
S1(config)# interface range f0/5 - 6
S1(config-if-range)# spanning-tree portfast
S1(config-if-range)# exit

! Habilitar BPDU Guard en los puertos conectados a las PCs (F0/6 en S1 y F0/18 en S2)
S1(config)# interface f0/6
S1(config-if)# spanning-tree bpduguard enable
S1(config-if)# exit

! O habilitar ambos globalmente para todos los puertos de acceso del switch:
S1(config)# spanning-tree portfast default
S1(config)# spanning-tree portfast bpduguard default
```
- **Verificación:** `show spanning-tree interface f0/6 detail` y `show spanning-tree summary`

### 6.5. Configurar SNMP en Switches y Routers Cisco (Módulo 2)
```ios
S1(config)# snmp-server community public RO
S1(config)# snmp-server community private RW
S1(config)# exit
S1# write memory
```

---

# PARTE II: MIKROTIK ROUTEROS (MÓDULOS 3, 4 Y 5)

## 7. Configuración Básica de MikroTik RouterOS (Módulo 3 - Lab 1)

### 7.1. Solución Completa Paso a Paso del Laboratorio 1 de MikroTik (CLI y WinBox)
```routeros
# 1. Cambiar la contraseña del usuario admin
/user set admin password=mikro123

# 2. Asignar Nombre (Identidad) al Router MikroTik (WinBox: System -> Identity)
/system identity set name=Mkt_01

# 3. Configurar Banner / Mensaje de Advertencia (WinBox: System -> Note)
/system note set show-at-login=yes note="ADVERTENCIA: ESTE EQUIPO ESTA SIENDO MONITOREADO POR NUESTRO NOC, CUALQUIER CAMBIO USTED ES RESPONSABLE!"

# 4. Renombrar las interfaces físicas ether1 (WAN) y ether2 (LAN)
/interface ethernet
set [ find default-name=ether1 ] name=WAN
set [ find default-name=ether2 ] name=LAN

# 5. Configurar DHCP Client en la interfaz WAN
/ip dhcp-client
add interface=WAN disabled=no

# 6. Asignar dirección IP a la interfaz LAN
/ip address
add address=192.168.0.1/24 interface=LAN network=192.168.0.0

# 7. Habilitar NAT (Masquerade) para salida a Internet
/ip firewall nat
add chain=srcnat out-interface=WAN action=masquerade

# 8. Configurar Servidor DHCP en la interfaz LAN
/ip pool
add name=dhcp_pool0 ranges=192.168.0.10-192.168.0.200

/ip dhcp-server
add address-pool=dhcp_pool0 interface=LAN name=dhcp1 lease-time=10m disabled=no

/ip dhcp-server network
add address=192.168.0.0/24 gateway=192.168.0.1 dns-server=8.8.8.8,8.8.4.4
```

### 7.2. Otras Configuraciones Esenciales del Módulo 3 (MTCNA)
```routeros
# Configurar DNS Caché
/ip dns set servers=8.8.8.8,8.8.4.4 allow-remote-requests=yes

# Ruta estática hacia otra red
/ip route add dst-address=192.168.20.0/24 gateway=192.168.0.2

# Crear Bridge y unir puertos LAN
/interface bridge add name=bridge-LAN
/interface bridge port add bridge=bridge-LAN interface=ether2
/interface bridge port add bridge=bridge-LAN interface=ether3

# Limitar ancho de banda con Simple Queue (64k Upload / 128k Download)
/queue simple add name="Limite-PC1" target=192.168.0.200/32 max-limit=64k/128k

# Backups (.backup binario vs .rsc texto plano)
/system backup save name=respaldo1
/export file=configuracion1
```

---

## 8. VPN IPsec Site-to-Site entre Routers MikroTik (Módulo 4)

### Configuración Completa en `Router-A` (LAN `10.90.1.0/24` -> Remota `192.168.11.0/24`)
```routeros
# FASE 1: Profile, Peer e Identity
/ip ipsec profile
add name=IPSEC1 hash-algorithm=sha1 enc-algorithm=aes-256 dh-group=modp1024 \
    proposal-check=obey lifetime=8h nat-traversal=yes dpd-interval=2m dpd-maximum-failures=5

/ip ipsec peer
add name=Cliente address=155.138.164.46 profile=IPSEC1 exchange-mode=main send-initial-contact=yes

/ip ipsec identity
add peer=Cliente auth-method=pre-shared-key secret="Curso!!123!!$"

# FASE 2: Proposal, Policy y Regla de Excepción NAT (NAT Bypass)
/ip ipsec proposal
add name=PRO-IPSEC1 auth-algorithms=sha1 enc-algorithms=aes-256-cbc lifetime=30m pfs-group=modp1024

/ip ipsec policy
add peer=Cliente tunnel=yes src-address=10.90.1.0/24 dst-address=192.168.11.0/24 \
    protocol=all action=encrypt level=require ipsec-protocols=esp proposal=PRO-IPSEC1

# Regla de NAT Bypass (DEBE IR EN LA POSICIÓN 0, ARRIBA DEL MASQUERADE)
/ip firewall nat
add chain=srcnat src-address=10.90.1.0/24 dst-address=192.168.11.0/24 action=accept \
    place-before=0 comment="Aceptar trafico VPN IPsec antes del Masquerade"

# Verificación desde el propio router
/ping 192.168.11.1 src-address=10.90.1.1
```

---

## 9. Seguridad en MikroTik y Filtrado Avanzado (Módulo 5)

### 9.1. Hardening y Protección contra Ataques al Router
```routeros
# 1. Detectar y Bloquear Escaneos de Puertos (Port Scanners - PSD)
/ip firewall filter
add chain=input protocol=tcp psd=21,3s,3,1 action=add-src-to-address-list \
    address-list="port-scanners" address-list-timeout=2w comment="Detectar Port Scanners"
add chain=input src-address-list="port-scanners" action=drop comment="Bloquear Port Scanners"

# 2. Mitigar Ataques DoS por límite de conexiones y Tarpit
/ip firewall filter
add chain=input protocol=tcp connection-limit=10,32 action=add-src-to-address-list \
    address-list=blocked-addr address-list-timeout=1d comment="Detectar exceso de conexiones TCP"
add chain=input protocol=tcp src-address-list=blocked-addr connection-limit=3,32 \
    action=tarpit comment="Tarpit a atacantes DoS"

# 3. Protección contra ataques SYN Flood y SYN Cookies
/ip firewall filter
add chain=forward protocol=tcp tcp-flags=syn connection-state=new \
    action=jump jump-target=SYN-Protect comment="SYN Flood protect"
add chain=SYN-Protect protocol=tcp tcp-flags=syn limit=400,5 connection-state=new action=accept
add chain=SYN-Protect protocol=tcp tcp-flags=syn connection-state=new action=drop
/ip settings set tcp-syncookies=yes

# 4. Evitar Ataques de Amplificación (Bloquear DNS/NTP/SNMP desde WAN)
/ip firewall filter
add chain=input in-interface=WAN protocol=udp dst-port=53,123,161 action=drop
```

### 9.2. Prácticas 1, 2 y 3 del Módulo 5
```routeros
# PRÁCTICA 1: Redirección de DNS y bloqueo por Address-List "restricted"
/ip firewall nat
add chain=dstnat protocol=tcp dst-port=53 action=redirect to-ports=53 comment="DNS"
add chain=dstnat protocol=udp dst-port=53 action=redirect to-ports=53
/ip firewall filter
add chain=forward dst-address-list=restricted action=drop

# PRÁCTICA 2: Bloquear Windows Update con TLS-Host
/ip firewall raw
add action=drop chain=prerouting comment="Blok Windows Update" protocol=tcp tls-host=windowsupdate.microsoft.com
add action=drop chain=prerouting comment="Blok Windows Update" protocol=tcp tls-host=download.microsoft.com
add action=drop chain=prerouting comment="Blok Windows Update" protocol=tcp tls-host=test.stats.update.microsoft.com
add action=drop chain=prerouting comment="Blok Windows Update" protocol=tcp tls-host=ntservicepack.microsoft.com
add action=drop chain=prerouting comment="Blok Windows Update" protocol=tcp tls-host=*.download.windowsupdate.com
add action=drop chain=prerouting comment="Blok Windows Update" protocol=tcp tls-host=*.update.microsoft.com
add action=drop chain=prerouting comment="Blok Windows Update" protocol=tcp tls-host=download.windowsupdate.com
add action=drop chain=prerouting comment="Blok Windows Update" protocol=tcp tls-host=*.windowsupdate.microsoft.com

# PRÁCTICA 3: Bloquear YouTube (Layer 7, Content y TLS)
# Opción 1: Layer 7
/ip firewall layer7-protocol
add name=Youtube regexp="^.+(youtube.com|googlevideo.com).*\$"
/ip firewall filter
add action=drop chain=forward layer7-protocol=Youtube

# Opción 2: Content
/ip firewall filter
add action=drop chain=forward content="youtube.com"
add action=drop chain=forward content="googlevideo.com"
add action=drop chain=forward content=".youtube."
add action=drop chain=forward content=".googlevideo."

# Opción 3: TLS Host
/ip firewall filter
add action=drop chain=forward protocol=tcp tls-host="youtube.com"
add action=drop chain=forward protocol=tcp tls-host="googlevideo.com"
add action=drop chain=forward protocol=tcp tls-host="*.youtube.*"
add action=drop chain=forward protocol=tcp tls-host="*.googlevideo.*"
```
