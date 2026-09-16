# Semana No. 9 - Ejercicio práctico (BGP)

## 1. TOPOLOGÍA Y DISPOSITIVOS

La topología simula la interconexión de tres Sistemas Autónomos (AS 100, AS 200, AS 300) mediante BGP como protocolo de enrutamiento exterior. Cada AS utiliza un IGP diferente internamente (OSPF, EIGRP, RIPv2), complementado con redundancia HSRP, agregación EtherChannel y conectividad WiFi.

### 1.1 Tabla de dispositivos y direccionamiento

| Dispositivo | Modelo    | Interfaz       | Dirección IP       | Función                           |
| ----------- | --------- | -------------- | ------------------ | --------------------------------- |
| R1          | 2911      | Gig0/0         | 192.168.1.2/24     | LAN AS 100 (HSRP Active)          |
| R1          | 2911      | Se0/0/0 (DCE)  | 10.0.12.1/30       | Enlace WAN hacia R2 (OSPF)        |
| R1          | 2911      | Se0/0/1        | 172.16.13.1/30     | Enlace eBGP hacia R3              |
| R2          | 2911      | Gig0/0         | 192.168.1.3/24     | LAN AS 100 (HSRP Standby)         |
| R2          | 2911      | Se0/0/0        | 10.0.12.2/30       | Enlace WAN hacia R1 (OSPF)        |
| R2          | 2911      | Se0/0/1        | 172.16.24.1/30     | Enlace eBGP hacia R4              |
| R3          | 2911      | Gig0/0         | 192.168.2.1/24     | LAN AS 200                        |
| R3          | 2911      | Se0/0/0 (DCE)  | 10.0.34.1/30       | Enlace WAN hacia R4 (EIGRP)       |
| R3          | 2911      | Se0/0/1 (DCE)  | 172.16.13.2/30     | Enlace eBGP hacia R1              |
| R4          | 2911      | Se0/0/0        | 10.0.34.2/30       | Enlace WAN hacia R3 (EIGRP)       |
| R4          | 2911      | Se0/0/1 (DCE)  | 172.16.24.2/30     | Enlace eBGP hacia R2              |
| R4          | 2911      | Se0/1/0 (DCE)  | 172.16.45.1/30     | Enlace eBGP hacia R5              |
| R5          | 2911      | Gig0/0         | 192.168.3.1/24     | LAN AS 300                        |
| R5          | 2911      | Se0/0/0        | 172.16.45.2/30     | Enlace eBGP hacia R4              |
| SW1         | 2960      | Fa0/2-3        | —                  | EtherChannel Po1 (LACP) hacia SW2 |
| SW2         | 2960      | Fa0/2-3        | —                  | EtherChannel Po1 (LACP) hacia SW1 |
| WRT300N     | WRT300N   | Internet (WAN) | 192.168.3.2/24     | Puerto WAN (GW: 192.168.3.1)      |
| WRT300N     | WRT300N   | LAN / WiFi     | 192.168.4.1/24     | Red inalámbrica (DHCP .100–.150)  |
| PC1         | PC-PT     | Fa0            | 192.168.1.10/24    | Cliente AS 100 (GW: 192.168.1.1)  |
| PC2         | PC-PT     | Fa0            | 192.168.1.11/24    | Cliente AS 100 (GW: 192.168.1.1)  |
| PC3         | PC-PT     | Fa0            | 192.168.2.10/24    | Cliente AS 200 (GW: 192.168.2.1)  |
| Laptop1     | Laptop-PT | WiFi (WPC300N) | DHCP (192.168.4.x) | Cliente inalámbrico AS 300        |

> **ℹ️ Notas sobre la topología:**
>
> - **HSRP:** R1 (activo, prioridad 110) y R2 (standby, prioridad 100) comparten la VIP 192.168.1.1 como gateway de la LAN en AS 100.
> - **iBGP:** R1↔R2 en AS 100 y R3↔R4 en AS 200, usando Loopbacks como update-source.
> - **Red inalámbrica:** SSID `LAB-BGP-WIFI`, seguridad WPA2 Personal, contraseña `Lab12345!`.

## 2. GUÍA TÉCNICA

Favor seguir los pasos de la [guía técnica](./2S2026_Ejemplo_Semana_9_a.pdf) de este mismo directorio.

## 3. PRUEBAS Y DEMOSTRACIÓN

### 3.1 Verificar IGP internos

1. En R1: `show ip ospf neighbor` → debe aparecer R2 con estado FULL
2. En R3: `show ip eigrp neighbors` → debe aparecer R4
3. En R5: `show ip route rip` → verificar rutas RIPv2 activas

### 3.2 Verificar BGP

1. En cualquier router: `show ip bgp summary` → todos los vecinos deben estar en estado **Established** (columna State con número)
2. `show ip bgp` → verificar que las redes de los tres AS aparezcan en la tabla BGP
3. `show ip route bgp` → confirmar rutas aprendidas por BGP marcadas con **B**

### 3.3 Verificar EtherChannel

1. En SW1: `show etherchannel summary` → Po1 debe mostrar flags (SU)
2. Verificar que Fa0/2 y Fa0/3 aparezcan como miembros activos (P)

### 3.4 Verificar HSRP

1. En R1: `show standby brief` → debe mostrar estado **Active**, VIP 192.168.1.1
2. En R2: `show standby brief` → debe mostrar estado **Standby**

### 3.5 Verificar conectividad extremo a extremo

1. Desde PC1: `ping 192.168.1.11` → PC2, misma LAN ✓
2. Desde PC1: `ping 192.168.2.10` → PC3, AS 100 → AS 200 vía BGP ✓
3. Desde PC1: `ping 192.168.3.1` → R5, AS 100 → AS 300 vía BGP ✓
4. Desde Laptop1: `ping 192.168.1.10` → PC1, WiFi AS 300 → AS 100 ✓
5. Desde PC1: `tracert 192.168.3.1` → observar saltos entre AS

## 4. COMANDOS DE TROUBLESHOOTING

### 4.1 Comandos generales

```
show running-config
show ip interface brief
show ip route
show ip bgp summary
show ip bgp
show standby brief
show etherchannel summary
```

### 4.2 Problemas comunes

| Problema                            | Solución                                                                                               |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Vecino BGP en estado Idle/Active    | Verificar IP y AS del neighbor, comprobar conectividad IP entre peers con `ping`                       |
| iBGP no establece sesión            | Verificar `update-source Loopback0` y que la Loopback sea alcanzable vía IGP                           |
| Rutas BGP no aparecen en tabla      | Verificar que la red exista en la tabla de enrutamiento antes de anunciarla con `network`              |
| HSRP no conmuta a Standby           | Verificar `preempt` configurado en ambos routers y que la VIP sea la misma                             |
| EtherChannel no se forma (flags SD) | Verificar mismo modo LACP (active/active) y misma configuración en ambos switches                      |
| Laptop no obtiene IP por WiFi       | Verificar DHCP habilitado en WRT300N, rango correcto (192.168.4.100–.150)                              |
| WiFi no alcanza otras redes         | Verificar ruta estática `ip route 192.168.4.0 255.255.255.0 192.168.3.2` en R5 y que se anuncie en BGP |
| Ping entre AS falla                 | Verificar `next-hop-self` en sesiones iBGP y que las rutas tengan next-hop alcanzable                  |
