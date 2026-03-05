.

🛡️ Implementación Integral de VPNs en Entornos Cisco (9 Escenarios)
Este repositorio contiene la documentación técnica, topologías y scripts de configuración para la implementación de 9 variantes de Redes Privadas Virtuales (VPN). El proyecto abarca desde tecnologías legacy hasta arquitecturas modernas de alta escalabilidad.

ID de Seguridad / PSK: 0331

Cifrado Estándar: AES-256 / SHA-256

📋 Índice de Escenarios
Site-to-Site Basada en Políticas (IKEv1)

Site-to-Site Basada en Políticas (IKEv2)

Site-to-Site VTI (IKEv1)

Site-to-Site VTI (IKEv2)

Túnel GRE sobre IPsec (IKEv1)

Túnel GRE sobre IPsec (IKEv2)

DMVPN Fase 2 (Dual-Hub/Spoke)

DMVPN Fase 3 con IKEv2 (Shortcuts)

Acceso Remoto L2TP/IPsec (Client-to-Site)

<a name="escenario-1"></a>

1. VPN Site-to-Site Basada en Políticas (IKEv1)
Objetivo: Conexión segura entre R1 y R2 sin interfaces lógicas, usando Crypto Maps para cifrar tráfico específico (ACL 100).

Autenticación: Pre-Shared Key 0331.

Comando de Verificación: show crypto isakmp sa (Estado: QM_IDLE).

<a name="escenario-2"></a>

2. VPN Site-to-Site Basada en Políticas (IKEv2)
Objetivo: Evolución del Escenario 1 hacia el framework IKEv2 para mayor eficiencia en la negociación.

Diferencia Clave: El crypto map invoca un ikev2-profile para gestionar identidades.

Comando de Verificación: show crypto ikev2 sa (Estado: READY).

<a name="escenario-3"></a>

3. VPN Site-to-Site VTI (IKEv1)
Objetivo: Uso de interfaces virtuales (Tunnel1) que eliminan el overhead de GRE, permitiendo enrutamiento directo.

Modo: tunnel mode ipsec ipv4.

Ventaja: Enrutamiento más limpio hacia la red remota vía interfaz lógica.

<a name="escenario-4"></a>

4. VPN Site-to-Site VTI (IKEv2)
Objetivo: Implementación de VTI con la robustez de IKEv2.

Seguridad: AES-256-CBC / SHA-256 / Grupo 5.

Comando de Verificación: show ip route (La red remota debe aparecer vía Tunnel1).

<a name="escenario-5"></a>

5. Túnel GRE sobre IPsec (IKEv1)
Objetivo: Transportar tráfico Multicast y protocolos de enrutamiento dinámico (EIGRP AS 1).

Configuración: IPsec protege el túnel GRE en mode transport.

Verificación: show ip eigrp neighbors sobre la interfaz Tunnel0.

<a name="escenario-6"></a>

6. Túnel GRE sobre IPsec (IKEv2)
Objetivo: Migración del túnel GRE a IKEv2.

Parámetros: Uso de Proposal, Policy y Keyring específicos de IKEv2.

Optimización: El modo transporte es vital para no duplicar cabeceras IP.

<a name="escenario-7"></a>

7. DMVPN Fase 2 (IKEv1)
Objetivo: Arquitectura Hub-and-Spoke que permite túneles directos Spoke-to-Spoke manteniendo el Next-Hop original.

NHRP: Los Spokes resuelven direcciones a través del Hub.

EIGRP: Requiere no ip next-hop-self en el Hub.

<a name="escenario-8"></a>

8. DMVPN Fase 3 con IKEv2
Objetivo: Optimización máxima con NHRP Redirect (Hub) y NHRP Shortcut (Spokes).

Funcionamiento: El Hub redirige el tráfico y los Spokes crean atajos dinámicos en la tabla de rutas.

Verificación: show ip route nhrp (Rutas marcadas con la letra H).

<a name="escenario-9"></a>

9. Acceso Remoto L2TP/IPsec (Client-to-Site)
Objetivo: Servidor VPN para clientes remotos (Windows Server 2012 R2) con autenticación MS-CHAPv2.

Configuración del Servidor (HUB-L2TP)
Bash
crypto isakmp policy 10
 encr aes 256
 hash sha
 authentication pre-share
 group 14  ! Vital para compatibilidad con Windows
crypto isakmp key 0331 address 0.0.0.0
!
crypto ipsec transform-set TS-L2TP esp-aes 256 esp-sha-hmac 
 mode transport
!
interface Virtual-Template1
 ip unnumbered GigabitEthernet0/0
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2
Configuración del Nodo ISP (Core de Transporte)
Bash
interface GigabitEthernet0/0
 ip address 20.23.3.2 255.255.255.252 ! Hacia HUB
interface GigabitEthernet0/1
 ip address 20.23.3.6 255.255.255.252 ! Hacia Cliente
ip cef
Análisis de Resultados y Troubleshooting
Validación de Fase 1: Exitosa. El debug muestra: SA has been authenticated with 20.23.3.5.

Validación de Fase 2: Exitosa. El log confirma: Successfully installed IPSEC SA.

Observación Técnica: El Error 651/789 reportado en el cliente Windows se determinó como una limitación del simulador ante el NAT-Traversal. No obstante, la infraestructura del router Cisco fue validada mediante sockets UDP (500/4500) y estados QM_IDLE.

🛠️ Herramientas de Verificación General
show crypto isakmp sa: Verifica Fase 1 (Management Plane).

show crypto ipsec sa: Verifica Fase 2 (Data Plane / Cifrado).

show ip nhrp: Muestra mapeos dinámicos en DMVPN.

show vpdn session: Monitorea usuarios conectados en L2TP.
