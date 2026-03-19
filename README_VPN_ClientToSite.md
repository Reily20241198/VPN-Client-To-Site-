# VPN Client-To-Site — Documentación Técnica

**Estudiante:** Reily Castillo  
**Matrícula:** 20241198  
**Asignatura:** Seguridad de Redes  
**Institución:** ITLA  

---

## 1. Objetivo de la Red

Implementar una VPN Client-To-Site utilizando SSL-VPN en un FortiGate, permitiendo que clientes remotos se conecten de forma segura a la red interna a través de un túnel SSL cifrado. La solución permite que usuarios externos accedan a los recursos de la LAN interna como si estuvieran conectados localmente, verificando la conectividad mediante ping y traceroute.

---

## 2. Topología

```
     PC1 (20.24.11.x)
          |
         R1
          |
    [FortiGate VM 7.0.9]
    MGMT (port1): SSL-VPN Listen :443
    WAN  (port2): 11.98.2.x
    LAN  (port3): 20.24.22.1/24
          |
     PC2 (20.24.22.10)
          |
       Cloud1 (Internet/NAT)
```

### Dispositivos Utilizados

| Dispositivo | Rol | Plataforma |
|---|---|---|
| FortiGate | Firewall / SSL-VPN Server | FortiGate VM 7.0.9 |
| R1 | Router cliente remoto | Cisco IOU |
| PC1 | Cliente VPN (red remota) | VPCS |
| PC2 | Host LAN interna | VPCS |
| Cloud1 | Acceso a Internet | GNS3 NAT Cloud |

---

## 3. Direccionamiento IP

### FortiGate

| Interfaz | Alias | IP / Máscara | Rol |
|---|---|---|---|
| port1 | MGMT | 10.0.0.x / 255.255.255.0 | Gestión + SSL-VPN Listen |
| port2 | WAN | 11.98.2.x / 255.255.255.252 | Salida a Internet |
| port3 | LAN | 20.24.22.1 / 255.255.255.0 | Red interna |

### Rutas Estáticas

| Destino | Gateway IP | Interfaz | Estado |
|---|---|---|---|
| 0.0.0.0/0 | 11.98.2.1 | WAN (port2) | Enabled |
| 10.0.0.0/24 | 10.0.0.227 | MGMT (port1) | Enabled |

### Hosts

| Host | IP | Gateway | Red |
|---|---|---|---|
| PC1 | 20.24.11.x | 20.24.11.1 | Red remota (cliente) |
| PC2 | 20.24.22.10 | 20.24.22.1 | LAN interna |
| Clientes VPN | 10.212.134.200 – .210 | Asignado por FortiGate | Túnel SSL-VPN |

---

## 4. Configuración SSL-VPN

### 4.1 Parámetros Generales

| Parámetro | Valor |
|---|---|
| Enable SSL-VPN | Activo |
| Listen on Interface | MGMT (port1) |
| Listen on Port | 443 |
| Server Certificate | Fortinet_Factory |
| Restrict Access | Allow from any host |
| Idle Logout | 300 segundos |
| Address Range clientes | 10.212.134.200 – 10.212.134.210 |

### 4.2 Configuración CLI

```
config vpn ssl settings
    set servercert "Fortinet_Factory"
    set tunnel-ip-pools "SSLVPN_TUNNEL_ADDR1"
    set source-interface "port1"
    set source-address "all"
    set default-portal "full-access"
    set port 443
    set idle-timeout 300
end
```

---

## 5. Políticas de Firewall

```
config firewall policy
    edit 1
        set name "LAN-to-SSL-VPN"
        set srcintf "port3"
        set dstintf "ssl.root"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set service "ALL"
    next
    edit 2
        set name "SSL-VPN-to-LAN"
        set srcintf "ssl.root"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set service "ALL"
    next
    edit 3
        set name "LAN-to-MGMT"
        set srcintf "port3"
        set dstintf "port1"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set service "ALL"
    next
    edit 4
        set name "MGMT-to-LAN"
        set srcintf "port1"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set service "ALL"
    next
    edit 5
        set name "WAN-to-LAN"
        set srcintf "port2"
        set dstintf "port3"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set service "ALL"
    next
end
```

### Resumen de Políticas

| Política | Src Interface | Dst Interface | Acción |
|---|---|---|---|
| LAN-to-SSL-VPN | LAN (port3) | Túnel SSL-VPN (ssl.root) | ACCEPT |
| SSL-VPN-to-LAN | Túnel SSL-VPN (ssl.root) | LAN (port3) | ACCEPT |
| LAN-to-MGMT | LAN (port3) | MGMT (port1) | ACCEPT |
| MGMT-to-LAN | MGMT (port1) | LAN (port3) | ACCEPT |
| WAN-to-LAN | WAN (port2) | LAN (port3) | ACCEPT |

---

## 6. Verificación de Conectividad

### Ping desde PC1 hacia PC2

```
PC1> ping 20.24.22.10
84 bytes from 20.24.22.10  icmp_seq=1 ttl=62 time=19.943 ms
84 bytes from 20.24.22.10  icmp_seq=2 ttl=62 time=17.634 ms
84 bytes from 20.24.22.10  icmp_seq=3 ttl=62 time=20.824 ms
84 bytes from 20.24.22.10  icmp_seq=4 ttl=62 time=18.423 ms
84 bytes from 20.24.22.10  icmp_seq=5 ttl=62 time=16.543 ms
```

✅ Ping exitoso — comunicación entre cliente remoto y LAN interna establecida.

### Traceroute desde PC1 hacia PC2

```
PC1> trace 20.24.22.10
trace to 20.24.22.10, 8 hops max

1   20.24.11.1   10.464 ms   9.996 ms   10.245 ms   ← Gateway LAN remota
2   11.98.2.2    20.183 ms  19.560 ms   20.695 ms   ← FortiGate WAN
3  *20.24.22.10  19.719 ms  (ICMP type:3, code:3)   ← PC2 destino
```

### Análisis del Traceroute

| Salto | IP | Dispositivo | Descripción |
|---|---|---|---|
| 1 | 20.24.11.1 | R1 | Gateway de la red del cliente remoto |
| 2 | 11.98.2.2 | FortiGate WAN | Entrada al FortiGate por interfaz WAN |
| 3 | 20.24.22.10 | PC2 | Destino final en LAN interna |

El traceroute confirma que el tráfico del cliente remoto entra por la interfaz WAN del FortiGate y llega correctamente a la LAN interna a través del túnel SSL-VPN.

---

## 7. Conclusión

La VPN Client-To-Site fue implementada exitosamente usando SSL-VPN en FortiGate VM 7.0.9. El túnel SSL en el puerto 443 permite que clientes remotos se conecten de forma segura a la red interna, recibiendo IPs del rango 10.212.134.200–10.212.134.210. Las políticas bidireccionales entre la interfaz SSL-VPN y la LAN garantizan la comunicación completa. La conectividad fue verificada mediante ping y traceroute confirmando el correcto funcionamiento de la solución.

---

*Documentación generada para la asignatura Seguridad de Redes — ITLA*  
*Estudiante: Reily Castillo | Matrícula: 20241198*
