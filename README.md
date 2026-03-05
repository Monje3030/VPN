# 🛡️ Implementación Integral de VPNs en Cisco (9 Escenarios)

Este repositorio contiene la documentación técnica, topologías y scripts de configuración para la implementación de 9 variantes de Redes Privadas Virtuales (VPN). El proyecto abarca desde tecnologías legacy hasta arquitecturas modernas de alta escalabilidad, utilizando como identificador de seguridad la **matrícula 0331**.

## 📊 Parámetros Globales de Seguridad
* **PSK (Pre-Shared Key):** `0331`
* **Cifrado Fase 1:** AES-256 / SHA-256
* **Cifrado Fase 2:** AES-256 / SHA-256
* **Grupo Diffie-Hellman:** Grupo 5 / Grupo 14

---

## 📋 Índice de Escenarios
1. [Site-to-Site Basada en Políticas (IKEv1)](#escenario-1)
2. [Site-to-Site Basada en Políticas (IKEv2)](#escenario-2)
3. [Site-to-Site VTI (IKEv1)](#escenario-3)
4. [Site-to-Site VTI (IKEv2)](#escenario-4)
5. [Túnel GRE sobre IPsec (IKEv1)](#escenario-5)
6. [Túnel GRE sobre IPsec (IKEv2)](#escenario-6)
7. [DMVPN Fase 2 (Multipunto)](#escenario-7)
8. [DMVPN Fase 3 con IKEv2](#escenario-8)
9. [Acceso Remoto L2TP/IPsec](#escenario-9)

---

<a name="escenario-1"></a>
## 1. Site-to-Site Basada en Políticas (IKEv1)
Establece una conexión segura entre R1 y R2 inspeccionando el tráfico mediante una ACL.
* **Método:** Policy-Based (Crypto Maps).
* **Verificación:** `show crypto isakmp sa` debe mostrar `QM_IDLE`.

[Image of Site-to-Site Policy-Based VPN architecture]

<a name="escenario-2"></a>
## 2. Site-to-Site Basada en Políticas (IKEv2)
Evolución hacia el framework IKEv2 para mayor eficiencia.
* **Clave:** Requiere `ikev2-profile` y `keyring`.
* **Verificación:** `show crypto ikev2 sa` en estado `READY`.

<a name="escenario-3"></a>
## 3. Site-to-Site VTI (IKEv1)
Uso de interfaces virtuales (`Tunnel1`) eliminando el overhead de GRE.
* **Modo:** `tunnel mode ipsec ipv4`.
* **Beneficio:** Permite enrutamiento directo hacia la interfaz de túnel.

<a name="escenario-4"></a>
## 4. Site-to-Site VTI (IKEv2)
VTI con seguridad superior.
* **Parámetros:** AES-256-CBC / SHA-256.
* **Ruta:** La red remota se alcanza vía `Tunnel1` en la tabla de ruteo.

<a name="escenario-5"></a>
## 5. Túnel GRE sobre IPsec (IKEv1)
Permite transportar protocolos de enrutamiento dinámico como **EIGRP AS 1**.
* **Modo:** IPsec en `mode transport`.
* **Verificación:** `show ip eigrp neighbors` sobre `Tunnel0`.

<a name="escenario-6"></a>
## 6. Túnel GRE sobre IPsec (IKEv2)
Migración de GRE a IKEv2 para mejor manejo de llaves.
* **Check:** El `sa-type` debe ser `transport` para optimizar la cabecera IP.

<a name="escenario-7"></a>
## 7. DMVPN Fase 2 (Multipunto)
Arquitectura Hub-and-Spoke con túneles directos Spoke-to-Spoke.
* **NHRP:** Los Spokes resuelven direcciones vía el Hub.
* **EIGRP:** Requiere `no ip next-hop-self` en el Hub.

<a name="escenario-8"></a>
## 8. DMVPN Fase 3 con IKEv2
Optimización con `NHRP Redirect` y `NHRP Shortcut`.
* **Resultado:** Los Spokes crean atajos dinámicos (Rutas `H` en `show ip route`).

---

<a name="escenario-9"></a>
## 9. Acceso Remoto L2TP/IPsec (Client-to-Site)
Implementación de servidor VPN para clientes Windows Server 2012 R2.

### Configuración del Servidor (HUB-L2TP)
```bash
# Fase 1: Compatibilidad con Windows (DH Group 14)
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 14
!
# Fase 2: Modo Transporte
crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac 
 mode transport
!
# Interfaz Virtual y Autenticación
interface Virtual-Template1
 ip unnumbered GigabitEthernet0/0
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2
