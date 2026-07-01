# 🌐 Route-Based Site-to-Site VPN using Cisco VTI and IKEv2

Este repositorio aloja las configuraciones listas para producción y la documentación de ingeniería para desplegar una **VPN basada en enrutamiento (Route-Based VPN)** nativa. Se utiliza **VTI (Virtual Tunnel Interface)** de Cisco eliminando por completo el encapsulado secundario de GRE y optimizando el uso de MTU/MSS.

La seguridad de las sesiones de control de Fase 1 se rige bajo los lineamientos robustos de **IKEv2**.

---

## 🗺️ Matriz de Direccionamiento IP (VLSM /24)

La red perimetral simula un entorno real donde las PCs se acoplan de forma directa a las interfaces LAN de los routers perimetrales `R2` y `R3` a través de interfaces GigabitEthernet físicas.

| Dispositivo / Interfaz | Dirección IP | Máscara de Red | Tipo / Propósito de Interfaz |
| :--- | :--- | :--- | :--- |
| **R1 (ISP) - G0/0** | 192.8.39.1 | 255.255.255.252 (/30) | Tránsito Público WAN (Sitio A) |
| **R1 (ISP) - G0/1** | 192.8.39.5 | 255.255.255.252 (/30) | Tránsito Público WAN (Sitio B) |
| **R2 (Sitio A) - G0/0** | 192.8.39.2 | 255.255.255.252 (/30) | IP Pública WAN Externa |
| **R2 (Sitio A) - G0/1** | 192.8.39.17 | 255.255.255.240 (/28) | Gateway LAN Local (PC1: .18) |
| **R3 (Sitio B) - G0/0** | 192.8.39.6 | 255.255.255.252 (/30) | IP Pública WAN Externa |
| **R3 (Sitio B) - G0/1** | 192.8.39.33 | 255.255.255.240 (/28) | Gateway LAN Local (PC2: .34) |
| **R2 (Sitio A) - Tunnel0** | 192.8.39.9 | 255.255.255.252 (/30) | Endpoint Lógico VTI Local |
| **R3 (Sitio B) - Tunnel0** | 192.8.39.10 | 255.255.255.252 (/30) | Endpoint Lógico VTI Remoto |

---

## 🛠️ Configuración de los Dispositivos (Cisco IOS)

```text
configure terminal
interface GigabitEthernet0/0
 ip address 192.8.39.1 255.255.255.252
 no shutdown
exit
interface GigabitEthernet0/1
 ip address 192.8.39.5 255.255.255.252
 no shutdown
exit

```

```text
configure terminal

interface GigabitEthernet0/0
 ip address 192.8.39.2 255.255.255.252
 no shutdown
exit

interface GigabitEthernet0/1
 ip address 192.8.39.17 255.255.255.240
 no shutdown
exit

ip route 0.0.0.0 0.0.0.0 192.8.39.1

! --- FASE 1 IKEv2 ---
crypto ikev2 proposal PROPUESTA-VTI
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy POLITICA-VTI
 proposal PROPUESTA-VTI
exit

crypto ikev2 keyring LLAVERO-VTI
 peer Router3
  address 192.8.39.6
  pre-shared-key ClaveVTI123
 exit
exit

crypto ikev2 profile PERFIL-IKEv2-VTI
 match identity remote address 192.8.39.6 255.255.255.255
 identity local address 192.8.39.2
 authentication local pre-share
 authentication remote pre-share
 keyring local LLAVERO-VTI
exit

! --- FASE 2 IPsec ---
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
exit

crypto ipsec profile PERFIL-VPN-VTI
 set transform-set TS-VTI
 set ikev2-profile PERFIL-IKEv2-VTI
exit

! --- INTERFAZ VTI NATIVA ---
interface Tunnel0
 ip address 192.8.39.9 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 192.8.39.6
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PERFIL-VPN-VTI
exit

! --- ENRUTAMIENTO BASADO EN RUTA ---
ip route 192.8.39.32 255.255.255.240 Tunnel0

```

```text
configure terminal

interface GigabitEthernet0/0
 ip address 192.8.39.6 255.255.255.252
 no shutdown
exit

interface GigabitEthernet0/1
 ip address 192.8.39.33 255.255.255.240
 no shutdown
exit

ip route 0.0.0.0 0.0.0.0 192.8.39.5

! --- FASE 1 IKEv2 ---
crypto ikev2 proposal PROPUESTA-VTI
 encryption aes-cbc-256
 integrity sha256
 group 14
exit

crypto ikev2 policy POLITICA-VTI
 proposal PROPUESTA-VTI
exit

crypto ikev2 keyring LLAVERO-VTI
 peer Router2
  address 192.8.39.2
  pre-shared-key ClaveVTI123
 exit
exit

crypto ikev2 profile PERFIL-IKEv2-VTI
 match identity remote address 192.8.39.2 255.255.255.255
 identity local address 192.8.39.6
 authentication local pre-share
 authentication remote pre-share
 keyring local LLAVERO-VTI
exit

! --- FASE 2 IPsec ---
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
exit

crypto ipsec profile PERFIL-VPN-VTI
 set transform-set TS-VTI
 set ikev2-profile PERFIL-IKEv2-VTI
exit

! --- INTERFAZ VTI NATIVA ---
interface Tunnel0
 ip address 192.8.39.10 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 192.8.39.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile PERFIL-VPN-VTI
exit

! --- ENRUTAMIENTO BASADO EN RUTA ---
ip route 192.8.39.16 255.255.255.240 Tunnel0

```

---

## 🔍 Comandos Estratégicos de Auditoría

Para verificar este modelo específico basado en rutas, las herramientas de diagnóstico recomendadas son:

1. **`show crypto ikev2 sa`**: Valida que la Fase 1 establezca el estado estable en **`READY`**.
2. **`show crypto ipsec sa`**: Observarás que `local ident` y `remote ident` muestran `0.0.0.0/0.0.0.0`. Esto verifica que el enrutamiento controla de forma libre el tráfico interesante inyectado en el túnel.

---

**Desarrollado por Zoe Daniela Bobonagua Acevedo** 🚀
