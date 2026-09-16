# Semana No. 9 - Ejercicio práctico (SSL/TLS)

## 1. TOPOLOGÍA Y DISPOSITIVOS

La topología simula un entorno administrativo simple donde se compara el acceso inseguro (HTTP y Telnet) frente al acceso seguro (HTTPS y SSH) en un router Cisco. El objetivo es demostrar, mediante capturas en modo simulación, cómo las credenciales viajan en texto plano con protocolos inseguros y de forma cifrada con SSL/TLS y SSH.

### 1.1 Tabla de dispositivos y direccionamiento

| Dispositivo  | Modelo    | Interfaz     | Dirección IP     | Función                                   |
| ------------ | --------- | ------------ | ---------------- | ----------------------------------------- |
| R1           | 2911      | Gig0/0       | 192.168.1.1/24   | Gateway LAN / Dispositivo administrado    |
| SW1          | 2960      | Fa0/1        | —                | Puerto hacia PC-Admin                     |
| SW1          | 2960      | Fa0/2        | —                | Puerto hacia Servidor-Web                 |
| SW1          | 2960      | Gig0/1       | —                | Uplink hacia R1 (Gig0/0)                  |
| PC-Admin     | PC-PT     | FastEthernet | 192.168.1.10/24  | Estación administradora (GW: 192.168.1.1) |
| Servidor-Web | Server-PT | FastEthernet | 192.168.1.100/24 | Servidor HTTP/HTTPS (GW: 192.168.1.1)     |

> **ℹ️ Notas sobre la topología:**
>
> - **Llaves RSA:** Se generan con `crypto key generate rsa modulus 1024` en Packet Tracer por limitaciones del simulador. En producción se recomienda **mínimo 2048 bits**, idealmente 4096.
> - **Dominio FQDN:** Se configura `ip domain-name redes2.local` en R1 como requisito previo para la generación de llaves RSA.
> - **Líneas VTY:** Se restringen con `transport input ssh` para rechazar cualquier intento de conexión por Telnet.
> - **Servidor-Web:** Expone simultáneamente HTTP (puerto 80) y HTTPS (puerto 443) para comparar capturas de tráfico.

## 2. GUÍA TÉCNICA

Favor seguir los pasos de la [guía técnica](./2S2026_Ejemplo_Semana_9_b.pdf) de este mismo directorio.

## 3. PRUEBAS Y DEMOSTRACIÓN

### 3.1 Verificar configuración básica de IP

1. En PC-Admin: `ping 192.168.1.1` → R1 debe responder
2. En PC-Admin: `ping 192.168.1.100` → Servidor-Web debe responder
3. En R1: `show ip interface brief` → Gig0/0 debe estar `up/up` con IP 192.168.1.1

### 3.2 Verificar SSH en R1

1. En R1: `show ip ssh` → debe mostrar **SSH Enabled - version 2.0**
2. En R1: `show crypto key mypubkey rsa` → debe mostrar la llave pública generada
3. En R1: `show ssh` → lista sesiones SSH activas
4. En R1: `show running-config | include username` → confirmar usuario `admin` con privilegio 15

### 3.3 Verificar acceso seguro (HTTPS y SSH)

1. Desde PC-Admin → Web Browser: `https://192.168.1.100` → debe cargar el sitio cifrado
2. Desde PC-Admin → Command Prompt: `ssh -l admin 192.168.1.1` → debe solicitar contraseña y entrar al CLI
3. Capturar paquetes en **modo Simulation** y verificar que los headers estén **cifrados**

### 3.4 Verificar rechazo de protocolos inseguros

1. Desde PC-Admin: `telnet 192.168.1.1` → la conexión debe ser **rechazada**
2. Desde PC-Admin → Web Browser: `http://192.168.1.1` → acceso web administrativo HTTP debe fallar

### 3.5 Comparación en modo simulación

1. Activar **Simulation Mode** (esquina inferior derecha de Packet Tracer)
2. Filtrar protocolos: dejar activos solo **HTTP, HTTPS, Telnet, SSH**
3. Repetir una conexión HTTP y una HTTPS hacia el servidor → abrir `Inbound PDU Details`
4. Comparar: en HTTP los headers viajan en **texto plano**, en HTTPS viajan **cifrados**
5. Repetir para Telnet vs SSH hacia R1 → observar la misma diferencia

## 4. COMANDOS DE TROUBLESHOOTING

### 4.1 Comandos generales

```
show running-config
show ip interface brief
show ip ssh
show crypto key mypubkey rsa
show ssh
show users
show ip http server status
show ip http server secure status
```

### 4.2 Problemas comunes

| Problema                                      | Solución                                                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `crypto key generate rsa` falla               | Verificar que `hostname` y `ip domain-name` estén configurados antes de generar las llaves               |
| SSH no conecta (Connection refused)           | Verificar `ip ssh version 2`, llaves RSA generadas y `transport input ssh` en las líneas VTY             |
| SSH pide contraseña pero la rechaza           | Confirmar usuario con `username admin privilege 15 secret ...` y `login local` en las líneas VTY         |
| Telnet sigue funcionando tras asegurar        | Revisar que `line vty 0 4` tenga `transport input ssh` (no `transport input all` ni `telnet`)            |
| HTTPS no carga en el navegador del PC-Admin   | Activar HTTPS en `Servidor-Web → Services → HTTP` y verificar conectividad IP con `ping 192.168.1.100`   |
| Capturas no muestran diferencia HTTP vs HTTPS | Verificar que los filtros de Simulation tengan habilitados HTTP y HTTPS y repetir la conexión desde cero |
| `show ip ssh` muestra versión 1.99            | Forzar versión 2 con `ip ssh version 2` en modo configuración global                                     |
| Llaves RSA no se regeneran                    | Eliminar las anteriores con `crypto key zeroize rsa` antes de ejecutar `crypto key generate rsa`         |
| Usuario admin no tiene acceso privilegiado    | Verificar `privilege 15` en la línea `username` y que la VTY use `login local`                           |
