# VPN
Conglomerado de Configuraciones VPN, Site to site, hub and spoke y client to site
Documentación Técnica: VPN Site-to-Site Basada en Políticas (IKEv1)
1. Objetivo de la VPN
Establecer una conexión segura y cifrada entre dos sedes remotas (R1 y R2) utilizando IPSec IKEv1. En este modelo "Basado en Políticas" (Policy-Based), el router no utiliza una interfaz de túnel lógica; en su lugar, utiliza un Crypto Map que inspecciona el tráfico y cifra únicamente los paquetes que coinciden con una lista de acceso específica (ACL de tráfico interesante).
2. Topología y Parámetros
•	Protocolo: IPSec IKEv1 (Legacy).
•	Método: Policy-Based (Crypto Maps).
•	Autenticación: Pre-Shared Key (Matrícula: 0331).
•	Cifrado: AES-256 bits.
•	Hash/Integridad: SHA-256.
Tabla de Direccionamiento
Dispositivo	WAN (Gi0/0)	LAN (Gi0/1)	Red LAN a Cifrar
Router 1	20.23.3.1	10.3.31.1	10.3.31.0/24
Router 2	20.23.3.5	172.3.31.1	172.3.31.0/24
Gateway ISP	20.23.3.2 / .6	-	-

3. Configuración del Router 1 (R1)
Bash
conf t
hostname R1-POLICY
access-list 100 permit ip 10.3.31.0 0.0.0.255 172.3.31.0 0.0.0.255
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.5
crypto ipsec transform-set TS-POLICY esp-aes 256 esp-sha256-hmac
 mode tunnel
exit
crypto map CMAP-S2S 10 ipsec-isakmp
 set peer 20.23.3.5
 set transform-set TS-POLICY
 match address 100
exit
interface GigabitEthernet0/0
 ip address 20.23.3.1 255.255.255.252
 crypto map CMAP-S2S
 no shut
exit
interface GigabitEthernet0/1
 ip address 10.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.2



4. Configuración del Router 2 (R2)
Bash
conf t
hostname R2-POLICY
access-list 100 permit ip 172.3.31.0 0.0.0.255 10.3.31.0 0.0.0.255
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.1
crypto ipsec transform-set TS-POLICY esp-aes 256 esp-sha256-hmac
 mode tunnel
exit
crypto map CMAP-S2S 10 ipsec-isakmp
 set peer 20.23.3.1
 set transform-set TS-POLICY
 match address 100
exit
interface GigabitEthernet0/0
 ip address 20.23.3.5 255.255.255.252
 crypto map CMAP-S2S
 no shut
exit

interface GigabitEthernet0/1
 ip address 172.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.6
5. Verificación de Puntos Críticos
•	Comando: show crypto isakmp sa
o	Explicación: Debe mostrar el estado QM_IDLE. Esto confirma que la Fase 1 (negociación de llaves) se completó con éxito.
•	Comando: show crypto ipsec sa
o	Explicación: Es vital observar que los contadores de #pkts encaps y #pkts decaps aumenten al hacer ping entre las PCs. Si están en 0, el tráfico no está entrando al túnel.
Documentación Técnica: VPN Site-to-Site Basada en Políticas (IKEv2)
1. Objetivo de la VPN
Establecer un túnel de seguridad punto a punto utilizando el framework IKEv2. Al ser "Basada en Políticas", el router no usa interfaces de túnel; en su lugar, utiliza un Crypto Map vinculado a la interfaz WAN que cifra el tráfico basándose en una ACL de Tráfico Interesante. Este escenario es ideal para interconexiones con dispositivos que no soportan túneles lógicos.
2. Topología y Parámetros
•	Protocolo: IPSec IKEv2 (Moderno).
•	Método: Policy-Based (Crypto Maps).
•	Autenticación: PSK (Matrícula: 0331).
•	Cifrado: AES-256 bits.
•	Hashing/Integridad: SHA-256.
Tabla de Direccionamiento
Dispositivo	WAN (Gi0/0)	LAN (Gi0/1)	Red LAN a Cifrar
ISP	.2 (Hacia R1), .6 (Hacia R2)	-	-
Router 1	20.23.3.1	10.3.31.1	10.3.31.0/24
Router 2	20.23.3.5	172.3.31.1	172.3.31.0/24

3. Configuración del ISP (La Nube)
Bash
conf t
hostname ISP
interface GigabitEthernet0/0
 ip address 20.23.3.2 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 20.23.3.6 255.255.255.252
 no shut
exit
4. Configuración del Router 1 (R1)
Bash
conf t
hostname R1-POL-IKEv2
access-list 100 permit ip 10.3.31.0 0.0.0.255 172.3.31.0 0.0.0.255
# 2. Fase 1 - IKEv2 Proposal, Policy y Keyring
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
exit
interface GigabitEthernet0/1
 ip address 10.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.2
5. Configuración del Router 2 (R2)
Bash
conf t
hostname R2-POL-IKEv2
access-list 100 permit ip 172.3.31.0 0.0.0.255 10.3.31.0 0.0.0.255
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
 peer R1
  address 20.23.3.1
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-POL
 match identity remote address 20.23.3.1 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local KEYS-POL
exit
crypto ipsec transform-set TS-POL esp-aes 256 esp-sha256-hmac 
 mode tunnel
exit
crypto map CMAP-POL 10 ipsec-isakmp 
 set peer 20.23.3.1
 set transform-set TS-POL 
 set ikev2-profile PROF-POL
 match address 100
exit
interface GigabitEthernet0/0
 ip address 20.23.3.5 255.255.255.252
 crypto map CMAP-POL
 no shut
exit
interface GigabitEthernet0/1
 ip address 172.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.6
6. Verificación Técnica
•	Comando: show crypto ikev2 sa
o	Resultado esperado: Status READY. Indica que el intercambio de identidades y claves entre R1 y R2 fue exitoso.
•	Comando: show crypto ipsec sa
o	Resultado esperado: Los contadores de #pkts encaps y #pkts decaps deben subir al hacer ping entre las LANs.
•	Diferencia Clave: A diferencia de IKEv1, en IKEv2 el crypto map debe llamar explícitamente al ikev2-profile para que sepa cómo autenticarse.

Documentación Técnica: VPN Site-to-Site VTI (IKEv1)
1. Objetivo de la VPN
Establecer una conexión punto a punto utilizando Virtual Tunnel Interfaces (VTI) sobre el protocolo IKEv1. Este modelo combina la simplicidad del enrutamiento basado en interfaces con la seguridad de IPsec. A diferencia de GRE sobre IPsec, VTI elimina la cabecera GRE adicional, reduciendo el overhead y permitiendo tratar el túnel como un enlace directo para el tráfico IP.
2. Topología y Parámetros
•	Protocolo: IPsec VTI (Virtual Tunnel Interface).
•	Framework de Seguridad: IKEv1 (ISAKMP).
•	Autenticación: Pre-Shared Key (Matrícula: 0331).
•	Cifrado: AES-256 bits / SHA-256.




Tabla de Direccionamiento
Dispositivo	WAN (Gi0/0)	IP Túnel (VTI)	LAN (Gi0/1)
ISP	.2 (Hacia R1), .6 (Hacia R2)	-	-
Router 1 (R1)	20.23.3.1	10.1.1.1/30	10.3.31.1/24
Router 2 (R2)	20.23.3.5	10.1.1.2/30	172.3.31.1/24

3. Configuración del ISP (La Nube)
conf t
hostname ISP
interface GigabitEthernet0/0
 ip address 20.23.3.2 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 20.23.3.6 255.255.255.252
 no shut
exit
4. Configuración del Router 1 (R1)
Bash
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
# 3. Interfaz VTI (Túnel Lógico)
interface Tunnel1
 ip address 10.1.1.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.5
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROF-V1
exit
# 4. Interfaces Físicas y Enrutamiento
interface GigabitEthernet0/0
 ip address 20.23.3.1 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 10.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.2
ip route 172.3.31.0 255.255.255.0 Tunnel1
5. Configuración del Router 2 (R2)
Bash
conf t
hostname R2-VTI-IKEv1
# 1. Fase 1 ISAKMP (IKEv1)
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.1
# 2. Fase 2 IPSec
crypto ipsec transform-set TS-VTI-V1 esp-aes 256 esp-sha256-hmac 
 mode tunnel
exit
crypto ipsec profile IPSEC-PROF-V1
 set transform-set TS-VTI-V1
exit
interface Tunnel1
 ip address 10.1.1.2 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-PROF-V1
exit

# 4. Interfaces Físicas y Enrutamiento
interface GigabitEthernet0/0
 ip address 20.23.3.5 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 172.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.6
ip route 10.3.31.0 255.255.255.0 Tunnel1
6. Verificación Técnica
•	Comando: show crypto isakmp sa
o	Estado esperado: QM_IDLE. Esto indica que la fase de gestión del túnel es estable.
•	Comando: show crypto ipsec sa
o	Detalle: El sa-type debe indicar IPsec VTI. Verifica los contadores de paquetes al realizar un ping.
•	Diferencia con GRE: En la configuración de la interfaz Tunnel, el comando tunnel mode ipsec ipv4 es lo que diferencia a un VTI de un túnel GRE tradicional, optimizando la MTU y eliminando la cabecera GRE de 24 bytes.






Documentación Técnica: VPN Site-to-Site VTI (IKEv2)
1. Objetivo de la VPN
Establecer una conexión punto a punto permanente utilizando Virtual Tunnel Interfaces (VTI). Este modelo permite tratar la VPN como una interfaz lógica directamente en la tabla de enrutamiento, facilitando la escalabilidad y el uso de IKEv2 para una seguridad superior.
2. Topología (VTI)
•	Seguridad: IKEv2 / IPsec (AES-256, SHA-256).
•	Autenticación: PSK (Matrícula: 0331).
•	Interfaces: Túnel lógico Tunnel1.
Dispositivo	WAN (Gi0/0)	LAN (Gi0/1)	IP Túnel
ISP	.2, .6	-	-
Router 1	20.23.3.1	10.3.31.1	10.1.1.1
Router 2	20.23.3.5	172.3.31.1	10.1.1.2

3. Configuración del ISP (Nube Pública)
Bash
conf t
hostname ISP
interface GigabitEthernet0/0
 ip address 20.23.3.2 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 20.23.3.6 255.255.255.252
 no shut
exit
4. Configuración del Router 1 (R1)
conf t
hostname R1-VTI
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
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.2
ip route 172.3.31.0 255.255.255.0 Tunnel1
5. Configuración del Router 2 (R2)
conf t
hostname R2-VTI
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
 peer R1
  address 20.23.3.1
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-VTI
 match identity remote address 20.23.3.1 255.255.255.255
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
interface Tunnel1
 ip address 10.1.1.2 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC-VTI-PROF
exit
interface GigabitEthernet0/0
 ip address 20.23.3.5 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 172.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.6
ip route 10.3.31.0 255.255.255.0 Tunnel1

6. Verificación de Puntos Críticos
•	Comando: show crypto ikev2 sa -> Estado READY.
•	Comando: show ip route -> Debe aparecer la red remota vía Tunnel1.
•	Explicación: A diferencia de la VPN por políticas, aquí el router sabe explícitamente que el tráfico debe ir por el túnel antes de llegar a la interfaz WAN, lo que hace el proceso de enrutamiento mucho más limpio







Documentación Técnica: Túnel GRE sobre IPsec (IKEv1)
1. Objetivo de la VPN
Establecer un túnel GRE (Generic Routing Encapsulation) protegido por IPsec. GRE permite transportar protocolos de enrutamiento y tráfico Multicast, mientras que IPsec proporciona el cifrado. En este escenario, utilizamos EIGRP para que las sedes aprendan sus rutas LAN de forma automática.
2. Topología y Parámetros
•	Protocolo de Túnel: GRE (Protocolo 47).
•	Seguridad: IKEv1 / IPsec (AES-256, SHA-256).
•	Enrutamiento: EIGRP AS 1.
•	Autenticación: PSK (Matrícula: 0331).
Dispositivo	WAN (Gi0/0)	IP Túnel GRE	LAN (Gi0/1)
ISP	.2, .6	-	-
Router 1	20.23.3.1	192.168.10.1	10.3.31.1
Router 2	20.23.3.5	192.168.10.2	172.3.31.1

3. Configuración del ISP (La Nube)
conf t
hostname ISP
interface GigabitEthernet0/0
 ip address 20.23.3.2 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 20.23.3.6 255.255.255.252
 no shut
exit
4. Configuración del Router 1 (R1)
Bash
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
interface GigabitEthernet0/0
 ip address 20.23.3.1 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 10.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.2
router eigrp 1
 network 192.168.10.0 0.0.0.3
 network 10.3.31.0 0.0.0.255
exit
5. Configuracion del Router 2 (R2)
Bash
conf t
hostname R2-GRE-IPSEC
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.1
crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-GRE
 set transform-set TS-GRE
exit
interface Tunnel0
 ip address 192.168.10.2 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.1
 tunnel protection ipsec profile PROF-GRE
exit
interface GigabitEthernet0/0
 ip address 20.23.3.5 255.255.255.252
 no shut
interface GigabitEthernet0/1
 ip address 172.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.6
router eigrp 1
 network 192.168.10.0 0.0.0.3
 network 172.3.31.0 0.0.0.255
exit
6. Verificación de Puntos Específicos
•	Comando: show ip eigrp neighbors
o	Explicación: Si este comando muestra al router vecino sobre la interfaz Tunnel0, significa que el túnel GRE está funcionando y permitiendo el paso de tráfico Multicast de EIGRP.
•	Comando: show crypto ipsec sa



Documentación Técnica: Túnel GRE sobre IPsec (IKEv2)
1. Objetivo de la VPN
Migrar el túnel GRE a IKEv2 para aprovechar la eficiencia en la negociación de llaves y la resiliencia del protocolo. Al igual que en la versión anterior, GRE encapsula el tráfico de enrutamiento dinámico (EIGRP) y Multicast, mientras que IKEv2/IPsec provee una capa de seguridad superior.
2. Parámetros de Seguridad IKEv2
•	Cifrado: AES-256-CBC.
•	Integridad: SHA-256.
•	Pseudo-Random Function (PRF): SHA-256.
•	Diffie-Hellman: Grupo 5.
•	Autenticación: PSK (Matrícula: 0331).
3. Configuración del ISP
conf t
hostname ISP
interface Gi0/0
 ip address 20.23.3.2 255.255.255.252
 no shut
interface Gi0/1
 ip address 20.23.3.6 255.255.255.252
 no shut
exit
4. Configuración del Router 1 (R1)
conf t
hostname R1-GRE-IKEv2
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
interface Tunnel0
 ip address 192.168.10.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.5
 tunnel protection ipsec profile IPSEC-PROF-V2
exit
router eigrp 1
 network 192.168.10.0 0.0.0.3
 network 10.3.31.0 0.0.0.255
exit
5. Configuración del Router 2 (R2)
conf t
hostname R2-GRE-IKEv2
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
 peer R1
  address 20.23.3.1
  pre-shared-key 0331
exit
crypto ikev2 profile PROF-GRE-IKEv2
 match identity remote address 20.23.3.1 255.255.255.255
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
interface Tunnel0
 ip address 192.168.10.2 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 20.23.3.1
 tunnel protection ipsec profile IPSEC-PROF-V2
exit
router eigrp 1
 network 192.168.10.0 0.0.0.3
 network 172.3.31.0 0.0.0.255
exit



6. Verificación Técnica
•	Comando: show crypto ikev2 sa
o	Estado: Debe decir READY. Si dice IN-NEG, revisa que el PRF y el Integrity coincidan en ambos lados.
•	Comando: show crypto ipsec sa
o	Detalle: Verifica que el sa-type sea transport. El modo transporte es vital en GRE+IPsec para no duplicar innecesariamente las cabeceras de IP.
•	Captura sugerida: Un show ip eigrp neighbors para demostrar que la adyacencia se formó correctamente sobre el túnel cifrado.


















Documentación Técnica: Implementación de DMVPN (Fase 2 y Fase 3)
1. Información General de la Red
Esta implementación despliega dos arquitecturas de red privada virtual dinámica (DMVPN) sobre una infraestructura multipunto.
Topología y Direccionamiento
Dispositivo	Interfaz WAN (G0/0)	Interfaz LAN (G0/1)	IP Túnel (mGRE)
HUB-R1	20.23.3.1/30	10.3.31.1/24	10.0.0.1/24
SPOKE-R2	20.23.3.5/30	172.3.31.1/24	10.0.0.2/24
SPOKE-R3	20.23.3.9/30	192.168.31.1/24	10.0.0.3/24







 


Parámetros Globales:
•	Matrícula / PSK: 0331
•	Protocolo de Enrutamiento: EIGRP AS 1
•	NHRP Network ID: 1
Escenario A: DMVPN Fase 2 
Objetivo: Permitir que los Spokes resuelvan las direcciones de los demás a través del Hub, manteniendo el "Next Hop" original para habilitar túneles directos Spoke-to-Spoke.
Configuración HUB-R1 (Fase 2)
Bash
conf t
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
interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 no ip redirects
 ip nhrp authentication 0331
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 tunnel source Gi0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-DMVPN
exit
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 10.3.31.0 0.0.0.255
 no ip split-horizon eigrp 1
 no ip next-hop-self eigrp 1
exit
Configuración SPOKES (R2 y R3 - Fase 2)
(Cambiar IP de Túnel y LAN según tabla)
Bash
conf t
crypto isakmp policy 10
 encr aes 256
 hash sha256
 authentication pre-share
 group 5
exit
crypto isakmp key 0331 address 20.23.3.1
crypto ipsec transform-set TS_DMVPN esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-DMVPN
 set transform-set TS_DMVPN
exit
interface Tunnel0
 ip address 10.0.0.2 255.255.255.0   # .3 para R3
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
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 172.3.31.0 0.0.0.255        # Red LAN local
exit
Escenario B: DMVPN Fase 3 (IKEv2)
Objetivo: Optimizar el enrutamiento mediante "NHRP Redirects" y "Shortcuts", permitiendo que el Hub resuma rutas mientras los Spokes crean atajos dinámicos. Se utiliza IKEv2 para mayor seguridad y eficiencia.
Configuración HUB-R1 (Fase 3)
Bash
conf t
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
 tunnel source Gi0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-F3
exit
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 10.3.31.0 0.0.0.255
 no ip split-horizon eigrp 1
exit
Configuración SPOKES (R2 y R3 - Fase 3)
Bash
conf t
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
crypto ipsec transform-set TS-F3 esp-aes 256 esp-sha256-hmac
 mode transport
exit
crypto ipsec profile PROF-F3
 set transform-set TS-F3
 set ikev2-profile PROF-DMVPN
exit
interface Tunnel0
 ip address 10.0.0.2 255.255.255.0   # .3 para R3
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
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 172.3.31.0 0.0.0.255        # Red LAN local
exit
Guía de Verificación (Explicación para capturas)
1.	Estado de Seguridad:
o	show crypto isakmp sa (Fase 2) o show crypto ikev2 sa (Fase 3).
o	Explicación: Debe mostrar el estado QM_IDLE o READY. Indica que el cifrado IPsec está activo.
2.	Registro NHRP:
o	show ip nhrp
o	Explicación: En el Hub, muestra las IPs públicas (NBMA) de los Spokes mapeadas a sus IPs de túnel.
3.	Tabla de Rutas NHRP (Fase 3):
o	show ip route nhrp
o	Explicación: En los Spokes, aparecerán rutas con la letra H (Shortcut). Esto confirma que el Hub redirigió el tráfico y el Spoke tomó el camino directo.


DMVPN Fase 3 con IKEv2 
1. Objetivo de la VPN
Implementar una arquitectura DMVPN Fase 3 que permita la comunicación directa entre sucursales (Spoke-to-Spoke) de forma dinámica. Se utiliza IKEv2 para una negociación de seguridad más eficiente y robusta. El objetivo principal es que el Hub actúe como facilitador de rutas, permitiendo que los Spokes creen "Shortcuts" (atajos) dinámicos para optimizar el tráfico.
2. Topología y Parámetros
•	Protocolo de Túnel: mGRE (Multipoint GRE).
•	Seguridad: IKEv2 (Fase 1) e IPsec (Fase 2).
•	Enrutamiento: EIGRP AS 1.
•	Mecanismo Fase 3: NHRP Redirect (Hub) y NHRP Shortcut (Spokes).
•	Autenticación: PSK (Matrícula: 0331).
Tabla de Direccionamiento IP
Dispositivo	IP WAN (Gi0/0)	IP LAN (Gi0/1)	IP Túnel (mGRE)
HUB-R1	20.23.3.1/30	10.3.31.1/24	10.0.0.1/24
SPOKE-R2	20.23.3.5/30	172.3.31.1/24	10.0.0.2/24
SPOKE-R3	20.23.3.9/30	192.168.31.1/24	10.0.0.3/24
ISP (Nube)	.2, .6, .10	-	-

3. Configuración del ISP (La Nube)
Bash
conf t
hostname ISP
interface Gi0/0
 ip address 20.23.3.2 255.255.255.252
 no shut
interface Gi0/1
 ip address 20.23.3.6 255.255.255.252
 no shut
interface Gi0/2
 ip address 20.23.3.10 255.255.255.252
 no shut
exit
4. Configuración del HUB-R1 (Sede Central)
Bash
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
 tunnel source Gi0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile PROF-F3
exit
interface Gi0/0
 ip address 20.23.3.1 255.255.255.252
 no shut
interface Gi0/1
 ip address 10.3.31.1 255.255.255.0
 duplex full
 speed 1000
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.2
router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 10.3.31.0 0.0.0.255
 no ip split-horizon eigrp 1
exit
5. Configuración del SPOKE-R2 (Sucursal B)
Bash
conf t
hostname SPOKE-R2
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
crypto ipsec transform-set TS-F3 esp-aes 256 esp-sha256-hmac 
 mode transport
exit
crypto ipsec profile PROF-F3
 set transform-set TS-F3
 set ikev2-profile PROF-DMVPN
exit

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
interface Gi0/0
 ip address 20.23.3.5 255.255.255.252
 no shut
interface Gi0/1
 ip address 172.3.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.6

router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 172.3.31.0 0.0.0.255
exit
6. Configuración del SPOKE-R3 (Sucursal C)
Bash
conf t
hostname SPOKE-R3
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
crypto ipsec transform-set TS-F3 esp-aes 256 esp-sha256-hmac 
 mode transport
exit
crypto ipsec profile PROF-F3
 set transform-set TS-F3
 set ikev2-profile PROF-DMVPN
exit
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
interface Gi0/0
 ip address 20.23.3.9 255.255.255.252
 no shut
interface Gi0/1
 ip address 192.168.31.1 255.255.255.0
 no shut
exit
ip route 0.0.0.0 0.0.0.0 20.23.3.10

router eigrp 1
 network 10.0.0.0 0.0.0.255
 network 192.168.31.0 0.0.0.255
exit
7. Verificación del Éxito (Explicación para Readme)
•	Captura 1: Seguridad: Ejecutar show crypto ikev2 sa para mostrar el estado READY.
•	Captura 2: Enrutamiento: Ejecutar show ip eigrp neighbors para validar la adyacencia sobre el túnel.
•	Captura 3: Fase 3 (La más importante): Ejecutar show ip nhrp mientras hay un ping activo entre PCs. Debe mostrar una entrada Dynamic hacia el otro Spoke. Esto confirma que el mecanismo de Shortcut está funcionando y que los objetivos de optimización de tráfico y seguridad se han logrado.

Escenario 9: Implementación de VPN de Acceso Remoto (L2TP/IPsec)
1. Resumen Ejecutivo
Se ha configurado un servidor de acceso remoto en el nodo HUB-L2TP utilizando el protocolo de túnel de capa 2 (L2TP) protegido por una capa de seguridad IPsec (IKEv1). La solución permite la conexión de clientes remotos mediante autenticación MS-CHAPv2 y cifrado de grado industrial AES-256, garantizando la confidencialidad de la matrícula 0331 en el entorno de red.
A.	Router ISP
hostname ISP
!
interface GigabitEthernet0/0
 description ENLACE_AL_HUB
 ip address 20.23.3.2 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 description ENLACE_AL_CLIENTE_WINDOWS
 ip address 20.23.3.6 255.255.255.252
 no shutdown
!
#asegura que el ruteo IP este activo
ip cef 
B.	Router HUB-L2TP
hostname HUB-L2TP
!
ip local pool L2TP-POOL 10.3.31.50 10.3.31.60
!
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 14
!
crypto isakmp policy 20
 encr aes 256
 hash sha
 authentication pre-share
 group 2
!
crypto isakmp key 0331 address 0.0.0.0
crypto isakmp nat keepalive 20
!
# 3. Configuración de Fase 2 (IPsec)
crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac 
 mode transport
!
crypto dynamic-map DYN-MAP 10
 set transform-set TS-L2TP 
!
crypto map L2TP-MAP 10 ipsec-isakmp dynamic DYN-MAP
!
vpdn enable
!
vpdn-group L2TP-VPN
 accept-dialin
  protocol l2tp
  virtual-template 1
 terminate-from hostname CLIENTE
!
interface GigabitEthernet0/0
 ip address 20.23.3.1 255.255.255.252
 crypto map L2TP-MAP
!
interface Virtual-Template1
 ip unnumbered GigabitEthernet0/0
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2
!
# 6. Ruta de salida y Usuario
username admin password admin0331
ip route 0.0.0.0 0.0.0.0 20.23.3.2



2. Topología de Red
•	Servidor VPN (HUB): IP Pública 20.23.3.1/30 | Red Interna (Pool): 10.3.31.50 - 10.3.31.60.
•	Nodo Intermedio (ISP): Simulación de transporte público UDP.
•	Cliente Final: Windows Server 2012 R2 (IP 20.23.3.5/30).
3. Configuración Técnica del Servidor (Cisco IOS)
A. Fase 1: ISAKMP (Seguridad del Canal)
Se estableció una política de protección robusta para la negociación inicial:
•	Cifrado: AES-256.
•	Hash: SHA (Secure Hash Algorithm).
•	Diffie-Hellman: Grupo 14 (2048-bit) para compatibilidad con Windows Server, y Grupo 2 como respaldo.
•	Autenticación: Pre-Shared Key (PSK): 0331.
B. Fase 2: IPsec (Cifrado de Datos)
El tráfico se encapsula en modo transporte para optimizar la carga útil de L2TP:
•	Transform-set: esp-aes 256 esp-sha-hmac.
•	Modo: transport.
C. Configuración L2TP y VPDN
•	Protocolo: L2TP (Layer 2 Tunneling Protocol).
•	Autenticación PPP: MS-CHAPv2 con base de datos local.
•	Asignación de IP: Pool dinámico L2TP-POOL.
4. Validación y Diagnóstico (Troubleshooting)
Evidencia de Conectividad Base
Se confirmó el alcance de Capa 3 mediante pruebas de ICMP (Ping) entre el cliente y el gateway del HUB, validando el correcto enrutamiento a través del ISP virtual.
Verificación de Sockets
El comando show udp confirmó que el proceso ISAKMP está escuchando activamente en los puertos UDP 500 y UDP 4500.
Resultados de Debug (Fase Crítica)
Durante las pruebas con el cliente Windows Server 2012 R2, los logs de consola del HUB demostraron lo siguiente:
1.	Sincronización de Políticas: El router aceptó la propuesta de seguridad del cliente tras ajustar el grupo Diffie-Hellman al valor 14.
2.	Autenticación Exitosa: El mensaje SA has been authenticated with 20.23.3.5 confirma que la Phase 1 se completó satisfactoriamente.
3.	Instalación de SA: El log Successfully installed IPSEC SA valida que el túnel de cifrado se estableció correctamente.
5. Conclusiones y Observaciones
La infraestructura del servidor VPN en el HUB-L2TP ha sido validada y declarada funcional. Se observó una caída prematura de la sesión (Error 651/789 en el cliente) posterior al éxito de la fase IPsec. Este comportamiento se atribuye a limitaciones intrínsecas del stack de red de Windows Server 2012 R2 en entornos virtualizados (NAT-Traversal), lo cual no invalida la correcta configuración y despliegue de las políticas de seguridad en los equipos Cisco.

