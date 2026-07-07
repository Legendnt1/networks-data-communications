# INFORME — VPN SITE-TO-SITE (IPSec)
## Proyecto MIEMPRESA · Red Multi-sede Perú · SI640

---

## 1. RESUMEN EJECUTIVO

Se implementó un túnel **VPN Site-to-Site IPSec** entre ROUTER_LIMA e ISP3
para habilitar **comunicación bidireccional** entre la red corporativa
interna y los servidores del sitio público, **sin eliminar el NAT overload**
existente.

### Problema que resuelve
El NAT overload (PAT) es unidireccional: los hosts internos pueden iniciar
conexiones hacia el exterior, pero el exterior NO puede iniciar hacia los
hosts internos. La VPN crea un canal cifrado que permite tráfico
bidireccional entre `10.192.0.0/16` y `200.100.50.0/24`, preservando las
IPs originales (sin traducción NAT).

---

## 2. ARQUITECTURA

```
[Hosts internos 10.192.0.0/16]
         |
    [ROUTER_LIMA] ==== Túnel IPSec (sobre 11.11.35.x) ==== [ISP3]
         |                                                    |
      [ISP1] ---- (transit: solo ve tráfico cifrado ESP) ---- [SW_SERVER]
                                                               |
                                                     [DNS .10 / WEB .20]
```

### Endpoints del túnel

| Extremo | Dispositivo | IP endpoint | Interfaz |
|---------|-------------|-------------|----------|
| Local | ROUTER_LIMA | 11.11.35.2 | Serial0/1/0 (vía ISP1) |
| Remoto | ISP3 | 11.11.35.10 | FastEthernet0/0 |

### Tráfico protegido
- **Red interna:** `10.192.0.0/16` (toda la corporativa, 5 países)
- **Red externa:** `200.100.50.0/24` (servidores públicos)

---

## 3. MATRIZ DE COMUNICACIÓN RESULTANTE

| Origen | Destino | Mecanismo | Resultado |
|--------|---------|-----------|-----------|
| Host interno | 200.100.50.x (servidores) | VPN (sin NAT) | Bidireccional |
| 200.100.50.x | Host interno | VPN | Bidireccional |
| Host interno | Internet genérico | NAT overload | Asimétrico |
| Internet genérico | Host interno | NAT overload | Bloqueado |

La VPN convierte la relación interno↔Site de asimétrica a **bidireccional**,
mientras el resto del tráfico a Internet sigue con NAT (seguridad perimetral
intacta).

---

## 4. PARÁMETROS IPSec

| Fase | Parámetro | Valor |
|------|-----------|-------|
| ISAKMP (Fase 1) | Encriptación | AES 256 |
| ISAKMP | Hash | SHA-1 |
| ISAKMP | Autenticación | pre-shared key |
| ISAKMP | Grupo Diffie-Hellman | **2** |
| ISAKMP | Lifetime | 86400 s |
| IPSec (Fase 2) | Transform set | ESP-AES-256 + ESP-SHA-HMAC |
| IPSec | Modo | tunnel (default) |
| PSK | Clave compartida | cisco123 |

---

## 5. CONFIGURACIÓN

### 5.1 Requisito previo — Licencia securityk9 (ROUTER_LIMA)

El Cisco 2911 requiere activar el feature de seguridad antes de usar
comandos crypto:

```cisco
license boot module c2900 technology-package securityk9
! (aceptar EULA con 'yes' — gratis en Packet Tracer, modo evaluación)
end
write memory
reload
```

Verificación tras reload:
```cisco
crypto ?
! debe listar: ipsec, isakmp, map, dynamic-map
```

### 5.2 NAT Exemption (ROUTER_LIMA)

Excluye el tráfico del túnel del NAT (para que viaje cifrado con IPs
originales), manteniendo NAT para el resto de Internet. Se usa ACL nombrada
(el 2911 de PT no soporta route-map con match interface):

```cisco
ip access-list extended NAT-INTERNET
 deny ip 10.192.0.0 0.0.255.255 200.100.50.0 0.0.0.255
 permit ip 10.192.0.0 0.0.255.255 any
!
no ip nat inside source list 1 interface Serial0/1/0 overload
no ip nat inside source list 1 interface Serial0/2/0 overload
!
ip nat inside source list NAT-INTERNET interface Serial0/1/0 overload
ip nat inside source list NAT-INTERNET interface Serial0/2/0 overload
```

La línea `deny` (primera) exime del NAT el tráfico al sitio público;
la `permit` mantiene NAT para todo lo demás.

### 5.3 IPSec en ROUTER_LIMA

```cisco
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
!
crypto isakmp key cisco123 address 11.11.35.10
!
crypto ipsec transform-set ESP-AES-SHA esp-aes 256 esp-sha-hmac
!
ip access-list extended VPN-LIMA-ISP3
 permit ip 10.192.0.0 0.0.255.255 200.100.50.0 0.0.0.255
!
crypto map CMAP-ISP3 10 ipsec-isakmp
 set peer 11.11.35.10
 set transform-set ESP-AES-SHA
 match address VPN-LIMA-ISP3
!
interface Serial0/1/0
 crypto map CMAP-ISP3
```

### 5.4 IPSec en ISP3

Configuración espejo: peer apunta a Lima, ACL invertida (origen local
primero), y ruta de retorno hacia la red corporativa.

```cisco
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
!
crypto isakmp key cisco123 address 11.11.35.2
!
crypto ipsec transform-set ESP-AES-SHA esp-aes 256 esp-sha-hmac
!
ip access-list extended VPN-ISP3-LIMA
 permit ip 200.100.50.0 0.0.0.255 10.192.0.0 0.0.255.255
!
crypto map CMAP-ISP3 10 ipsec-isakmp
 set peer 11.11.35.2
 set transform-set ESP-AES-SHA
 match address VPN-ISP3-LIMA
!
interface FastEthernet0/0
 crypto map CMAP-ISP3
!
ip route 10.192.0.0 255.255.0.0 11.11.35.9
```

La ruta estática `10.192.0.0/16 → 11.11.35.9` es CRÍTICA: sin ella el túnel
sube pero el tráfico de retorno no encuentra camino a los hosts internos.

---

## 6. ORDEN DE PROCESAMIENTO (ROUTER_LIMA, Serial0/1/0)

### Tráfico saliente (interno → Site)
```
1. NAT (route/ACL) → deny para VPN → NO traduce
2. Crypto map → matchea VPN-LIMA-ISP3 → CIFRA (ESP)
3. Sale por Serial0/1/0 cifrado
```

### Tráfico saliente (interno → Internet genérico)
```
1. NAT → permit → TRADUCE a 11.11.35.2
2. Crypto map → no matchea → sale sin cifrar (normal)
```

### Tráfico entrante (Site → interno)
```
1. Paquete ESP cifrado llega → IOS lo detecta y DESCIFRA
2. Paquete original recuperado (200.100.50.x → 10.192.x.x)
3. Routing interno entrega al host destino
```

NAT y crypto coexisten sin conflicto: NAT actúa en Se0/1/0 y Se0/2/0
(hacia ISPs), crypto solo en Se0/1/0 (hacia el Site).

---

## 7. BUGS RESUELTOS DURANTE LA IMPLEMENTACIÓN

| # | Problema | Solución |
|---|----------|----------|
| 1 | 2911 sin comandos crypto | Activar securityk9 + reload |
| 2 | `route-map ... match interface` inválido para NAT | ACL nombrada NAT-INTERNET |
| 3 | `mode tunnel` rechazado | Omitirlo (es el default) |
| 4 | `MM_NO_STATE` en negociación ISAKMP | group 5 → **group 2** (PT no negocia DH5) |
| 5 | ISP3 con peer/ACL de Lima (sin invertir) | Peer 11.11.35.2 + ACL espejo |
| 6 | `clear crypto` no existe en PT | Quitar/reaplicar crypto map en interfaz |
| 7 | Túnel "frío" tras rebote de enlace | Disparar con tráfico interno (ping desde PC) |

---

## 8. VERIFICACIÓN

### 8.1 Disparar el túnel (desde adentro, modo Realtime)

El túnel se levanta solo con "tráfico interesante" originado desde la red
interna:

```
! Desde PC1_VENTAS_LIMA:
ping 200.100.50.10
! (primeros 1-2 fallan por negociación, resto responde)
```

### 8.2 Verificar SA activa (ROUTER_LIMA)

```cisco
show crypto isakmp sa
! debe mostrar: 11.11.35.10  11.11.35.2  QM_IDLE  ACTIVE

show crypto ipsec sa
! debe mostrar: #pkts encaps > 0, #pkts decaps > 0
!               current outbound spi: <valor distinto de 0x0>
```

### 8.3 Test bidireccional (el objetivo del proyecto)

```
! Desde DNS_PUBLICO (200.100.50.10):
ping 10.192.42.166      ! MAIL_LIMA → RESPONDE
ping 10.192.40.10       ! PC Ventas Lima → RESPONDE
ping 10.192.57.138      ! PC Puno (vía Lima) → RESPONDE
```

Si el sitio público alcanza los hosts internos, la bidireccionalidad
está lograda.

### 8.4 Confirmar que NAT sigue activo para Internet

```cisco
show ip nat translations
! debe mostrar traducciones para tráfico NO-VPN
! NO debe haber entradas hacia 200.100.50.x (ese va por VPN, sin NAT)
```

---

## 9. NOTA — VPN INTERNACIONAL DESCARTADA

Inicialmente se diseñó una VPN cascada entre los 4 routers de país
(Lima↔Quito↔Bogotá, Lima↔SCH↔BA). Se **descartó** porque:

- Los países no tienen LANs internas (solo routers), por lo que la crypto
  ACL solo cubría IPs de tránsito `10.192.36.x`.
- Esto rompía la conectividad básica entre routers (el tráfico intentaba
  cifrarse sin túnel establecido → se descartaba).

Los crypto maps y ACLs de la cascada internacional fueron eliminados. La
única VPN site-to-site operativa es **Lima ↔ ISP3**, con propósito real
(bidireccionalidad interno↔Site).

---

## 10. JUSTIFICACIÓN DE INGENIERÍA

La arquitectura NAT + VPN coexistiendo combina lo mejor de ambos:

- **NAT overload:** oculta los hosts internos del descubrimiento desde
  Internet, ahorra IPs públicas (una IP para toda la red), y mantiene la
  seguridad perimetral (el exterior no inicia hacia interno).
- **VPN IPSec:** provee un canal cifrado y bidireccional para el tráfico
  específico interno↔Site, sin exponer la infraestructura ni sacrificar
  el NAT.

Los servidores públicos pueden iniciar conexiones hacia hosts internos
—operación normalmente bloqueada por NAT— únicamente para tráfico que
matchea las políticas IPSec, proveyendo un canal seguro y controlado.