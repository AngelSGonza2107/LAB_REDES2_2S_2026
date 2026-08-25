# Semana No. 7 - Ejercicio práctico

## Descripción

Topología realizada en **Cisco Packet Tracer** para practicar:

- Creación y propagación de VLANs mediante **VTP**.
- Redundancia de enlaces con **EtherChannel (LACP)**.
- Enrutamiento inter-VLAN mediante **router-on-a-stick**.
- Enrutamiento dinámico con **OSPF**.
- Configuración de servicios **DNS** y **HTTP**.
- Implementación de una red inalámbrica con **WPA2** y **DHCP**.

## Topología

![Topología de red](../assets/sem7_topologia.png)

## Dispositivos

| Dispositivo         | Cantidad | Función principal                    |
| ------------------- | -------: | ------------------------------------ |
| Router 2911         |        2 | Enrutamiento inter-VLAN y OSPF       |
| Switch 2960         |        3 | VLANs, VTP, enlaces troncales y LACP |
| Router WRT300N      |        1 | Acceso inalámbrico, WPA2 y DHCP      |
| Server-PT           |        2 | Servicios DNS y HTTP                 |
| PC-PT               |        2 | Hosts de las VLAN 10 y 20            |
| Laptop / Smartphone |        2 | Clientes inalámbricos de prueba      |

## Plan de direccionamiento

| Segmento                 | Red                  | Gateway        | Dispositivos principales     |
| ------------------------ | -------------------- | -------------- | ---------------------------- |
| VLAN 10 - Administración | `192.168.10.0/24`    | `192.168.10.1` | PC1 `192.168.10.10`          |
| VLAN 20 - Ventas         | `192.168.20.0/24`    | `192.168.20.1` | PC2 `192.168.20.10`          |
| VLAN 30 - Servidores     | `192.168.30.0/24`    | `192.168.30.1` | DNS `.10`, HTTP `.20`        |
| VLAN 99 - Nativa         | Sin direccionamiento | -              | Troncales                    |
| Enlace R1-R2             | `10.0.0.0/30`        | -              | R1 `.1`, R2 `.2`             |
| R2-WR1                   | `172.16.0.0/24`      | `172.16.0.1`   | WR1 WAN `172.16.0.2`         |
| WLAN                     | `172.16.1.0/24`      | `172.16.1.1`   | DHCP `172.16.1.100` a `.150` |

## Enlaces principales

| Origen      | Destino      | Uso                             |
| ----------- | ------------ | ------------------------------- |
| SW1 Fa0/1-3 | SW2 Fa0/1-3  | EtherChannel 1                  |
| SW1 Fa0/4-6 | SW3 Fa0/1-3  | EtherChannel 2                  |
| SW2 Fa0/4   | SW3 Fa0/4    | Enlace troncal                  |
| SW1 Gi0/1   | R1 Gi0/0     | Troncal para router-on-a-stick  |
| R1 Gi0/1    | R2 Gi0/0     | Enlace WAN                      |
| R2 Gi0/1    | WR1 Internet | Acceso hacia la red inalámbrica |

## 1. VLANs y VTP

### SW1 - Servidor VTP

```cisco
enable
configure terminal
hostname SW1
vtp mode server
vtp domain SEM7-REDES
vtp password cisco123
vtp version 2

vlan 10
 name ADMINISTRACION
vlan 20
 name VENTAS
vlan 30
 name SERVIDORES
vlan 99
 name NATIVA
end
write memory
```

### SW2 y SW3 - Clientes VTP

Cambiar el `hostname` según el switch que se esté configurando.

```cisco
enable
configure terminal
hostname SW2
vtp mode client
vtp domain SEM7-REDES
vtp password cisco123
vtp version 2
end
write memory
```

Puertos de acceso:

```cisco
! SW2
interface FastEthernet0/20
 switchport mode access
 switchport access vlan 10
interface FastEthernet0/10
 switchport mode access
 switchport access vlan 30

! SW3
interface FastEthernet0/20
 switchport mode access
 switchport access vlan 20
interface FastEthernet0/10
 switchport mode access
 switchport access vlan 30
```

## 2. EtherChannel y enlaces troncales

### Port-Channel 1 entre SW1 y SW2

```cisco
interface range FastEthernet0/1-3
 channel-group 1 mode active
interface Port-channel1
 switchport mode trunk
 switchport trunk native vlan 99
```

### Port-Channel 2 entre SW1 y SW3

En SW1:

```cisco
interface range FastEthernet0/4-6
 channel-group 2 mode active
interface Port-channel2
 switchport mode trunk
 switchport trunk native vlan 99
```

En SW3:

```cisco
interface range FastEthernet0/1-3
 channel-group 2 mode active
interface Port-channel2
 switchport mode trunk
 switchport trunk native vlan 99
```

Enlaces troncales adicionales:

```cisco
! SW2 y SW3 - enlace entre switches
interface FastEthernet0/4
 switchport mode trunk
 switchport trunk native vlan 99

! SW1 - enlace hacia R1
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk native vlan 99
```

## 3. Enrutamiento

### R1 - Router-on-a-stick y OSPF

```cisco
enable
configure terminal
hostname R1

interface GigabitEthernet0/0
 no shutdown
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
interface GigabitEthernet0/0.99
 encapsulation dot1Q 99 native

interface GigabitEthernet0/1
 ip address 10.0.0.1 255.255.255.252
 no shutdown

router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.20.0 0.0.0.255 area 0
 network 192.168.30.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
end
write memory
```

### R2 - Interfaces y OSPF

```cisco
enable
configure terminal
hostname R2

interface GigabitEthernet0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/1
 ip address 172.16.0.1 255.255.255.0
 no shutdown

router ospf 1
 network 10.0.0.0 0.0.0.3 area 0
 network 172.16.0.0 0.0.0.255 area 0
end
write memory
```

## 4. Servicios y red inalámbrica

| Configuración | Valor                                                       |
| ------------- | ----------------------------------------------------------- |
| Servidor DNS  | `192.168.30.10/24`, gateway `192.168.30.1`                  |
| Servidor HTTP | `192.168.30.20/24`, gateway `192.168.30.1`                  |
| Registros DNS | `www.sem7redes.com` y `sem7redes.com` hacia `192.168.30.20` |
| WR1 WAN       | `172.16.0.2/24`, gateway `172.16.0.1`                       |
| WR1 LAN       | `172.16.1.1/24`                                             |
| SSID          | `LAB-WIFI`                                                  |
| Seguridad     | WPA2 Personal, AES                                          |
| Contraseña    | `Sem7Redes2026`                                             |
| DHCP          | Desde `172.16.1.100`, máximo 50 usuarios                    |

En el servidor HTTP se deben activar los servicios **HTTP/HTTPS**. Los clientes inalámbricos deben conectarse al SSID indicado, autenticarse con WPA2 y obtener su dirección mediante DHCP.

## 5. Verificación

| Prueba                               | Resultado esperado                      |
| ------------------------------------ | --------------------------------------- |
| VLANs visibles en SW2 y SW3          | VLAN 10, 20, 30 y 99 propagadas por VTP |
| Estado de EtherChannel               | Port-Channel 1 y 2 activos con LACP     |
| Vecindad OSPF                        | R1 y R2 en estado adyacente             |
| Ping de PC1 a PC2                    | Responde                                |
| Ping desde Laptop1 a `192.168.30.20` | Responde                                |
| `nslookup www.sem7redes.com`         | Resuelve `192.168.30.20`                |
| Acceso a `http://www.sem7redes.com`  | Muestra la página del servidor HTTP     |

Comandos de verificación:

```cisco
show vtp status
show vlan brief
show interfaces trunk
show etherchannel summary
show interfaces port-channel 1
show ip interface brief
show ip ospf neighbor
show ip route ospf
show ip route
```

Desde los equipos finales:

```text
ipconfig
ping 192.168.30.20
ping 192.168.10.10
nslookup www.sem7redes.com
```
