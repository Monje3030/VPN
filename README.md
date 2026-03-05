
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
## Topologia utilizada en Laboratorios 1,2,3,4,5, & 6
<img width="1156" height="530" alt="image" src="https://github.com/user-attachments/assets/5c1008de-57e4-49f6-b2de-eba4645b4b2a" />

<a name="escenario-1"></a>
## 1. Site-to-Site Basada en Políticas (IKEv1) (Router 1 y Router 2)
Establece una conexión segura entre R1 y R2 inspeccionando el tráfico mediante una ACL.
```
conf t
hostname R1-POL-IKEv2
access-list 100 permit ip 10.3.31.0 0.0.0.255 172.3.31.0 0.0.0.255
crypto ikev2 proposal PROP-POL
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 5
exit
crypto ikev2 policy POL-POL
 proposal PROP-POL
exit
crypto ikev2 keyring KEYS-POL
 peer R2
  address 20.23.3.5
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-POL
 match identity remote address 20.23.3.5 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS-POL
exit
crypto ipsec transform-set TS-POL esp-aes 256 esp-sha256-hmac
 mode tunnel
exit
crypto map CMAP-POL 10 ipsec-isakmp
 set peer 20.23.3.5
 set transform-set TS-POL
 set ikev2-profile PROF-POL
 match address 100
exit
interface GigabitEthernet0/0
 ip address 20.23.3.1 255.255.255.252
 crypto map CMAP-POL
no shut
ip route 0.0.0.0 0.0.0.0 20.23.3.2
```

* **Método:** Policy-Based (Crypto Maps).
* **Verificación:** `show crypto isakmp sa` debe mostrar `QM_IDLE`.

<a name="escenario-2"></a>
## 2. Site-to-Site Basada en Políticas (IKEv2) (Router 1 & Router 2)
Evolución hacia el framework IKEv2 para mayor eficiencia.
```
conf t
hostname R1-POL-IKEv2
access-list 100 permit ip 10.3.31.0 0.0.0.255 172.3.31.0 0.0.0.255
crypto ikev2 proposal PROP-POL
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 5
exit
crypto ikev2 policy POL-POL
 proposal PROP-POL
exit
crypto ikev2 keyring KEYS-POL
 peer R2
  address 20.23.3.5
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-POL
 match identity remote address 20.23.3.5 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS-POL
exit
crypto ipsec transform-set TS-POL esp-aes 256 esp-sha256-hmac
 mode tunnel
exit
crypto map CMAP-POL 10 ipsec-isakmp
 set peer 20.23.3.5
 set transform-set TS-POL
 set ikev2-profile PROF-POL
 match address 100
exit
interface GigabitEthernet0/0
 ip address 20.23.3.1 255.255.255.252
 crypto map CMAP-POL
no shut
ip route 0.0.0.0 0.0.0.0 20.23.3.2
```
* **Clave:** Requiere `ikev2-profile` y `keyring`.
* **Verificación:** `show crypto ikev2 sa` en estado `READY`.

<a name="escenario-3"></a>
## 3. Site-to-Site VTI (IKEv1) (Router 1 & Router 2)
Uso de interfaces virtuales (`Tunnel1`) eliminando el overhead de GRE.
* **Modo:** `tunnel mode ipsec ipv4`.
* **Beneficio:** Permite enrutamiento directo hacia la interfaz de túnel.
```bash
conf t
hostname R1-VTI-IKEv1
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.5
crypto ipsec transform-set TS-VTI-V1 esp-aes 256 esp-sha256-hmac
 mode tunnel
exit
crypto ipsec profile IPSEC-PROF-V1
 set transform-set TS-VTI-V1
exit
interface Tunnel1
 ip address 10.1.1.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.5
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROF-V1
exit
ip route 172.3.31.0 255.255.255.0 Tunnel1
```

<a name="escenario-4"></a>
## 4. Site-to-Site VTI (IKEv2) (Router 1 & Router 2)
VTI con seguridad superior.
* **Parámetros:** AES-256-CBC / SHA-256.
* **Ruta:** La red remota se alcanza vía `Tunnel1` en la tabla de ruteo.
```
conf t
hostname R1-VTI-IKEv2
!
crypto ikev2 proposal PROP-VTI
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 5
exit
crypto ikev2 policy POL-VTI
 proposal PROP-VTI
exit
crypto ikev2 keyring KEY-VTI
 peer R2
  address 20.23.3.5
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-VTI
 match identity remote address 20.23.3.5 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEY-VTI
exit
crypto ipsec transform-set TS-VTI esp-aes 256 esp-sha256-hmac
 mode tunnel
exit
crypto ipsec profile IPSEC-VTI-PROF
 set transform-set TS-VTI
 set ikev2-profile PROF-VTI
exit
!
interface Tunnel1
 ip address 10.1.1.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.5
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-VTI-PROF
exit
interface GigabitEthernet0/0
 ip address 20.23.3.1 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 10.3.31.1 255.255.255.0
 no shut
ip route 0.0.0.0 0.0.0.0 20.23.3.2
ip route 172.3.31.0 255.255.255.0 Tunnel1
```

<a name="escenario-5"></a>
## 5. Túnel GRE sobre IPsec (IKEv1) (Router 1 & Router 2)
Permite transportar protocolos de enrutamiento dinámico como **EIGRP AS 1**.
* **Modo:** IPsec en `mode transport`.
* **Verificación:** `show ip eigrp neighbors` sobre `Tunnel0`.
```
conf t
hostname R1-GRE-IPSEC
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.5
crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-GRE
 set transform-set TS-GRE
exit
interface Tunnel0
 ip address 192.168.10.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.5
 tunnel protection ipsec profile PROF-GRE
exit
router eigrp 1
 network 192.168.10.0 0.0.0.3
 network 10.3.31.0 0.0.0.255
```


<a name="escenario-6"></a>
## 6. Túnel GRE sobre IPsec (IKEv2) (Router 1 & Router 2)
Migración de GRE a IKEv2 para mejor manejo de llaves.
* **Check:** El `sa-type` debe ser `transport` para optimizar la cabecera IP.
```
conf t
hostname R1-GRE-IKEv2
!
crypto ikev2 proposal PROP-GRE
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 5
exit
crypto ikev2 policy POL-GRE
 proposal PROP-GRE
exit
crypto ikev2 keyring KEYS-GRE
 peer R2
  address 20.23.3.5
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-GRE-IKEv2
 match identity remote address 20.23.3.5 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS-GRE
exit
crypto ipsec transform-set TS-GRE-V2 esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile IPSEC-PROF-V2
 set transform-set TS-GRE-V2
 set ikev2-profile PROF-GRE-IKEv2
exit
!
interface Tunnel0
 ip address 192.168.10.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.5
 tunnel protection ipsec profile IPSEC-PROF-V2
exit
router eigrp 1
 network 192.168.10.0 0.0.0.3
 network 10.3.31.0 0.0.0.255
```
## Topologia utilizada en Laboratorios 7 y 8

<img width="1200" height="564" alt="image" src="https://github.com/user-attachments/assets/4dd2c7c5-c1fa-4799-9582-9e469dc788df" />

<a name="escenario-7"></a>
## 7. DMVPN Fase 2 (Multipunto) (Router HUB)
Arquitectura Hub-and-Spoke con túneles directos Spoke-to-Spoke.
* **NHRP:** Los Spokes resuelven direcciones vía el Hub.
* **EIGRP:** Requiere `no ip next-hop-self` en el Hub.
``` HUB R1
conf t
hostname HUB-R1-F2
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 0.0.0.0
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-DMVPN
 set transform-set TS_DMVPN
exit
!
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 no ip redirects
 ip nhrp authentication 0331
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
exit
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 10.3.31.0 0.0.0.255
 no ip split-horizon eigrp 1
 no ip next-hop-self eigrp 1
```
``` Spoke R2
conf t
hostname SPOKE-R2-F2
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.1
!
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-DMVPN
 set transform-set TS_DMVPN
exit
!
interface Tunnel0
 ip address 10.0.0.2 255.255.255.0
 no ip redirects
 ip nhrp authentication 0331
 ip nhrp map 10.0.0.1 20.23.3.1
 ip nhrp map multicast 20.23.3.1
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 1
 tunnel source Gi0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
exit
!
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 172.3.31.0 0.0.0.255
exit
```
``` Spoke R3
conf t
hostname SPOKE-R3-F2
!
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.1
!
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-DMVPN
 set transform-set TS_DMVPN
exit
!
interface Tunnel0
 ip address 10.0.0.3 255.255.255.0
 no ip redirects
 ip nhrp authentication 0331
 ip nhrp map 10.0.0.1 20.23.3.1
 ip nhrp map multicast 20.23.3.1
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 1
 tunnel source Gi0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
exit
!
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 192.168.31.0 0.0.0.255
exit
```
<a name="escenario-8"></a>
## 8. DMVPN Fase 3 con IKEv2 (Router HUB)
Optimización con `NHRP Redirect` y `NHRP Shortcut`.
* **Resultado:** Los Spokes crean atajos dinámicos (Rutas `H` en `show ip route`).
```HUB R1 F3
conf t
hostname HUB-R1
crypto ikev2 proposal PROP-0331
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 5
exit
crypto ikev2 policy POL-0331
 proposal PROP-0331
exit
crypto ikev2 keyring KEYS-DMVPN
 peer ALL-SPOKES
  address 0.0.0.0 0.0.0.0
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-DMVPN
 match identity remote address 0.0.0.0
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS-DMVPN
exit
crypto ipsec transform-set TS-F3 esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-F3
 set transform-set TS-F3
 set ikev2-profile PROF-DMVPN
exit
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 no ip redirects
 ip nhrp authentication 0331
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 ip nhrp redirect
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-F3
exit
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 10.3.31.0 0.0.0.255
 no ip split-horizon eigrp 1
```

```Spoke R2 Fase 3
conf t
hostname SPOKE-R2-F3
!
crypto ikev2 proposal PROP-0331
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 5
exit
crypto ikev2 policy POL-0331
 proposal PROP-0331
exit
crypto ikev2 keyring KEYS-DMVPN
 peer HUB
  address 20.23.3.1
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-DMVPN
 match identity remote address 20.23.3.1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS-DMVPN
exit
!
crypto ipsec transform-set TS-F3 esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-F3
 set transform-set TS-F3
 set ikev2-profile PROF-DMVPN
exit
!
interface Tunnel0
 ip address 10.0.0.2 255.255.255.0
 no ip redirects
 ip nhrp authentication 0331
 ip nhrp map 10.0.0.1 20.23.3.1
 ip nhrp map multicast 20.23.3.1
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 1
 ip nhrp shortcut
 tunnel source Gi0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-F3
exit
!
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 172.3.31.0 0.0.0.255
exit
```

``` Spoke R3 Fase 3
conf t
hostname SPOKE-R3-F3
!
crypto ikev2 proposal PROP-0331
 encryption aes-cbc-256
 integrity sha256
 prf sha256
 group 5
exit
crypto ikev2 policy POL-0331
 proposal PROP-0331
exit
crypto ikev2 keyring KEYS-DMVPN
 peer HUB
  address 20.23.3.1
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-DMVPN
 match identity remote address 20.23.3.1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS-DMVPN
exit
!
crypto ipsec transform-set TS-F3 esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-F3
 set transform-set TS-F3
 set ikev2-profile PROF-DMVPN
exit
!
interface Tunnel0
 ip address 10.0.0.3 255.255.255.0
 no ip redirects
 ip nhrp authentication 0331
 ip nhrp map 10.0.0.1 20.23.3.1
 ip nhrp map multicast 20.23.3.1
 ip nhrp nhs 10.0.0.1
 ip nhrp network-id 1
 ip nhrp shortcut
 tunnel source Gi0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-F3
exit
!
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 192.168.31.0 0.0.0.255
exit
```
## Topologia utilzada

<img width="749" height="320" alt="image" src="https://github.com/user-attachments/assets/81d39af6-1b7b-43cf-b2c5-a33e0124abb9" />

<a name="escenario-9"></a>
## 9. Acceso Remoto L2TP/IPsec (Client-to-Site) (Router HUB)
Implementación de servidor VPN para clientes Windows Server 2012 R2.

### Configuración del Servidor (HUB-L2TP)
```
conf t
hostname HUB-L2TP
ip local pool L2TP-POOL 10.3.31.50 10.3.31.60
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 14
exit
crypto isakmp policy 20
 encr aes 256
 hash sha
 authentication pre-share
 group 2
exit
crypto isakmp key 0331 address 0.0.0.0
crypto isakmp nat keepalive 20
crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac
 mode transport
exit
crypto dynamic-map DYN-MAP 10
 set transform-set TS-L2TP
exit
crypto map L2TP-MAP 10 ipsec-isakmp dynamic DYN-MAP
vpdn enable
vpdn-group L2TP-VPN
 accept-dialin
 protocol l2tp
 virtual-template 1
 terminate-from hostname CLIENTE
exit
interface GigabitEthernet0/0
 ip address 20.23.3.1 255.255.255.252
 crypto map L2TP-MAP
no shut
interface Virtual-Template1
 ip unnumbered GigabitEthernet0/0
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2
exit
username admin password admin0331
ip route 0.0.0.0 0.0.0.0 20.23.3.2
```
## 10. Enlace a Video demostrativo
https://youtu.be/nKbeAdw5-mw
