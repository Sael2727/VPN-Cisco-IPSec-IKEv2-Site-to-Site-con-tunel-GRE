# 🔐 VPN Cisco IPSec IKEv2 Site-to-Site — Túnel GRE

<div align="center">

![Cisco](https://img.shields.io/badge/Cisco-IOS-blue?style=for-the-badge&logo=cisco)
![IPSec](https://img.shields.io/badge/IPSec-IKEv2-green?style=for-the-badge)
![GRE](https://img.shields.io/badge/Tunnel-GRE-purple?style=for-the-badge)
![Platform](https://img.shields.io/badge/Platform-PNETLab-orange?style=for-the-badge)
![License](https://img.shields.io/badge/Uso-Educativo-red?style=for-the-badge)

**Sael Germán García** | Matrícula: `2025-0725`  
Asignatura: Seguridad de Redes | Profesor: Jonathan Rondón  
Instituto Tecnológico de las Américas — ITLA | 2026

</div>

---

## 📋 Descripción

Configuración y verificación de una **VPN Cisco IPSec IKEv2 Site-to-Site con túnel GRE** para permitir comunicación segura entre dos LANs remotas a través de un router ISP que simula la red pública. **GRE** crea el túnel lógico punto a punto entre R1 y R2, mientras **IPSec con IKEv2** cifra el tráfico GRE entre las direcciones WAN de ambos peers.

> 💡 **Clave técnica:** La ACL de IPSec no apunta directamente a las LANs, sino al tráfico **GRE** entre las IPs WAN (10.7.25.1 ↔ 10.7.25.5). GRE corresponde al protocolo IP **47**, visible en `show crypto ipsec sa`. La diferencia respecto al lab anterior (GRE + IKEv1) es el uso de **IKEv2** con su estructura de proposal, policy, keyring y profile en lugar del `crypto isakmp policy`.

---

## 🗺️ Topología de Red

La topología conserva dos routers como peers VPN, un router ISP al centro y una LAN en cada extremo. El ISP no cifra tráfico, solo brinda alcance entre R1 y R2.

### 📊 Direccionamiento IP

| Dispositivo | Interfaz | IP / Máscara | Función |
|:-----------:|:--------:|:-------------:|---------|
| R1 | Ethernet0/0 | 10.7.25.1/30 | WAN hacia ISP |
| ISP | Ethernet0/0 | 10.7.25.2/30 | Enlace hacia R1 |
| ISP | Ethernet0/1 | 10.7.25.6/30 | Enlace hacia R2 |
| R2 | Ethernet0/0 | 10.7.25.5/30 | WAN hacia ISP |
| R1 | Ethernet0/1 | 10.7.25.65/27 | Gateway LAN-A |
| VPC1 | eth0 | 10.7.25.66/27 | Cliente LAN-A |
| R2 | Ethernet0/1 | 10.7.25.97/27 | Gateway LAN-B |
| VPC2 | eth0 | 10.7.25.98/27 | Cliente LAN-B |
| **R1** | **Tunnel0** | **10.7.25.141/30** | **Extremo GRE R1** |
| **R2** | **Tunnel0** | **10.7.25.142/30** | **Extremo GRE R2** |

---

## ⚙️ Parámetros de la VPN

| Parámetro | Valor |
|:---------:|-------|
| Tipo de VPN | Site-to-Site punto a punto con túnel GRE |
| Versión IKE | IKEv2 |
| Cifrado IKEv2 | AES-CBC-256 |
| Integridad IKEv2 | SHA256 |
| Grupo Diffie-Hellman | Grupo 5 |
| Autenticación | Pre-Shared Key |
| Clave compartida | VPN12345 |
| Transform-set IPSec | TS-IKEV2: esp-aes 256 esp-sha256-hmac |
| Crypto map | VPN-MAP |
| Tráfico protegido | GRE entre 10.7.25.1 y 10.7.25.5 |
| Ruta R1 → LAN-B | 10.7.25.96/27 vía Tunnel0 |
| Ruta R2 → LAN-A | 10.7.25.64/27 vía Tunnel0 |

---

## 🔍 Funcionamiento

GRE crea una interfaz virtual **Tunnel0** que funciona como enlace lógico punto a punto entre R1 y R2, permitiendo transportar tráfico de una LAN a la otra. IPSec con **IKEv2** protege ese tráfico GRE entre las direcciones WAN.

La ACL `VPN-TRAFFIC` identifica el protocolo GRE (IP 47) entre las IPs WAN como el tráfico interesante. El `crypto map VPN-MAP` se aplica a la interfaz WAN `Ethernet0/0` y activa la negociación IKEv2 cuando se detecta ese tráfico. Las rutas estáticas envían el tráfico de cada LAN hacia `Tunnel0`.

---

## 🚀 Scripts de Configuración

### ISP
```cisco
hostname ISP
interface Ethernet0/0
 description ENLACE_A_R1
 ip address 10.7.25.2 255.255.255.252
 no shutdown
interface Ethernet0/1
 description ENLACE_A_R2
 ip address 10.7.25.6 255.255.255.252
 no shutdown
```

### R1 — Configuración Principal
```cisco
hostname R1
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.1 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_A
 ip address 10.7.25.65 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.2

crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5
crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROP
crypto ikev2 keyring IKEV2-KEYRING
 peer R2
  address 10.7.25.5
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345
crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 10.7.25.5 255.255.255.255
 identity local address 10.7.25.1
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING

crypto ipsec transform-set TS-IKEV2 esp-aes 256 esp-sha256-hmac
 mode tunnel

ip access-list extended VPN-TRAFFIC
 permit gre host 10.7.25.1 host 10.7.25.5

crypto map VPN-MAP 10 ipsec-isakmp
 description PROTEGER_TRAFICO_GRE_IKEV2_HACIA_R2
 set peer 10.7.25.5
 set transform-set TS-IKEV2
 set pfs group5
 set ikev2-profile IKEV2-PROFILE
 match address VPN-TRAFFIC

interface Ethernet0/0
 crypto map VPN-MAP

interface Tunnel0
 description TUNEL_GRE_IKEV2_HACIA_R2
 ip address 10.7.25.141 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.7.25.5
 tunnel mode gre ip
 no shutdown

ip route 10.7.25.96 255.255.255.224 Tunnel0
```

### R2 — Configuración Principal
```cisco
hostname R2
interface Ethernet0/0
 description WAN_HACIA_ISP
 ip address 10.7.25.5 255.255.255.252
 no shutdown
interface Ethernet0/1
 description LAN_B
 ip address 10.7.25.97 255.255.255.224
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.7.25.6

crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256
 integrity sha256
 group 5
crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROP
crypto ikev2 keyring IKEV2-KEYRING
 peer R1
  address 10.7.25.1
  pre-shared-key local VPN12345
  pre-shared-key remote VPN12345
crypto ikev2 profile IKEV2-PROFILE
 match identity remote address 10.7.25.1 255.255.255.255
 identity local address 10.7.25.5
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEV2-KEYRING

crypto ipsec transform-set TS-IKEV2 esp-aes 256 esp-sha256-hmac
 mode tunnel

ip access-list extended VPN-TRAFFIC
 permit gre host 10.7.25.5 host 10.7.25.1

crypto map VPN-MAP 10 ipsec-isakmp
 description PROTEGER_TRAFICO_GRE_IKEV2_HACIA_R1
 set peer 10.7.25.1
 set transform-set TS-IKEV2
 set pfs group5
 set ikev2-profile IKEV2-PROFILE
 match address VPN-TRAFFIC

interface Ethernet0/0
 crypto map VPN-MAP

interface Tunnel0
 description TUNEL_GRE_IKEV2_HACIA_R1
 ip address 10.7.25.142 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 10.7.25.1
 tunnel mode gre ip
 no shutdown

ip route 10.7.25.64 255.255.255.224 Tunnel0
```

### Configuración de VPCs
```bash
# VPC1
ip 10.7.25.66 255.255.255.224 10.7.25.65

# VPC2
ip 10.7.25.98 255.255.255.224 10.7.25.97
```

---

## ✅ Verificación del Túnel

```cisco
show ip interface brief
show running-config interface tunnel0
show running-config | section crypto
show access-lists
show interface tunnel0
show crypto ikev2 sa
show crypto ipsec sa
show crypto session
```

| Comando | Estado esperado |
|:-------:|------------------|
| `show interface tunnel0` | up/up — protocolo de transporte GRE/IP |
| `show access-lists` | Matches en ACL GRE confirmados |
| `show crypto ikev2 sa` | READY |
| `show crypto ipsec sa` | Protocolo 47 (GRE), encaps/decaps activos |
| `show crypto session` | UP-ACTIVE |

> 💡 **Nota sobre traceroute:** El mensaje "Destination port unreachable" al final del trace es normal en VPCS — usa paquetes UDP y el host destino responde que el puerto no está disponible. El ping exitoso y las sesiones crypto activas confirman que los hosts fueron alcanzados correctamente.

---

## 📊 Resultados

| Prueba | Resultado |
|:------:|:---------:|
| Tunnel0 en R1 y R2 | ✅ up/up |
| Protocolo de transporte del túnel | ✅ GRE/IP |
| ACL VPN-TRAFFIC (GRE entre IPs WAN) | ✅ Matches confirmados en R1 y R2 |
| Rutas hacia LAN remota vía Tunnel0 | ✅ Presentes en ambos routers |
| IKEv2 SA | ✅ READY |
| IPSec SA | ✅ Protocolo 47, encaps/decaps activos |
| Sesión crypto | ✅ UP-ACTIVE |
| Ping VPC1 → VPC2 | ✅ Exitoso |
| Ping VPC2 → VPC1 | ✅ Exitoso |

---

## 📁 Archivos del Repositorio

| Archivo | Descripción |
|:-------:|-------------|
| [`SaelGerman_2025-0725_Script_GRE_IKEV2.txt`](SaelGerman_2025-0725_Script_GRE_IKEV2.txt) | Scripts de configuración Cisco IOS |
| [`SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-con-tunel-GRE_P2.pdf`](SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-con-tunel-GRE_P2.pdf) | Documentación técnica completa |

---

## 🖼️ Capturas de Pantalla

- 📸 [Figura 1 — Topología VPN GRE + IPSec IKEv2](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/01_topologia_gre_ikev2.png)
- 📸 [Figura 2 — Interfaces del ISP](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/02_isp_show_ip_interface_brief.png)
- 📸 [Figura 3 — Interfaces de R1](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/03_r1_show_ip_interface_brief.png)
- 📸 [Figura 4 — Configuración de Tunnel0 en R1](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/04_r1_show_running_config_interface_tunnel0.png)
- 📸 [Figura 5 — Parámetros crypto IKEv2/IPSec en R1](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/05_r1_show_running_config_section_crypto.png)
- 📸 [Figura 6 — ACL VPN-TRAFFIC en R1 (GRE hacia R2)](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/06_r1_show_access_lists.png)
- 📸 [Figura 7 — Ruta de R1 hacia LAN remota por Tunnel0](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/07_r1_show_ip_route_lan_remota.png)
- 📸 [Figura 8 — Interfaces de R2](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/08_r2_show_ip_interface_brief.png)
- 📸 [Figura 9 — Configuración de Tunnel0 en R2](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/09_r2_show_running_config_interface_tunnel0.png)
- 📸 [Figura 10 — Parámetros crypto IKEv2/IPSec en R2](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/10_r2_show_running_config_section_crypto.png)
- 📸 [Figura 11 — Ruta de R2 hacia LAN remota por Tunnel0](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/11_r2_show_ip_route_lan_remota.png)
- 📸 [Figura 12 — Estado de Tunnel0 up/up y transporte GRE/IP](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/12_r2_show_interface_tunnel0.png)
- 📸 [Figura 13 — Prueba desde VPC1 hacia VPC2](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/13_vpc1_show_ip_ping_trace_to_vpc2.png)
- 📸 [Figura 14 — Prueba desde VPC2 hacia VPC1](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/14_vpc2_show_ip_ping_trace_to_vpc1.png)
- 📸 [Figura 15 — Estado IKEv2 SA en R1 (READY)](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/15_r1_show_crypto_ikev2_sa.png)
- 📸 [Figura 16 — Estado IKEv2 SA en R2 (READY)](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/16_r2_show_crypto_ikev2_sa.png)
- 📸 [Figura 17 — IPSec SA en R1 (encaps/decaps activos)](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/17_r1_show_crypto_ipsec_sa.png)
- 📸 [Figura 18 — IPSec SA en R2 (encaps/decaps activos)](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/18_r2_show_crypto_ipsec_sa.png)
- 📸 [Figura 19 — Sesión crypto activa en R1 (UP-ACTIVE)](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/19_r1_show_crypto_session.png)
- 📸 [Figura 20 — Sesión crypto activa en R2 (UP-ACTIVE)](SaelGerman_2025-0725_Capturas_GRE_IKEV2_GitHub/20_r2_show_crypto_session.png)

---

## 📎 Recursos

📄 **Documentación Técnica:** [Ver Informe PDF](SaelGerman_2025-0725_VPN-IPSec-IKEv2-Site-to-Site-con-tunel-GRE_P2.pdf)  
▶️ **Video Demostración:** [Ver en YouTube](https://youtu.be/L7wljJoUwUA)

---

## 📚 Referencias

1. Cisco Systems. *Configuring a GRE Tunnel over IPSec with IKEv2*. Documentación oficial Cisco IOS.
2. Reconocimiento especial: Troubleshooting, base del script y documentación apoyado en Inteligencia Artificial.

---

<div align="center">

*Este laboratorio fue desarrollado exclusivamente con fines académicos y educativos.*

</div>
