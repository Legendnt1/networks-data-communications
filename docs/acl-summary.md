# INFORME — POLÍTICAS DE SEGURIDAD ACL
## Proyecto MIEMPRESA

---

## 1. RESUMEN EJECUTIVO

Se implementaron **11 políticas de seguridad** distribuidas en la red corporativa:
- **1 Política Empresarial/Intersede** (ROUTER_LIMA)
- **5 Políticas de Seguridad IP** (una por sede, todas distintas)
- **5 Políticas de Seguridad de Servicios** (una por sede, todas distintas)

Cada política responde a un vector de seguridad diferente, sin repetición de lógica entre sedes.

### Mapa general de políticas

| Sede | Política IP | Política de Servicios |
|------|-------------|----------------------|
| Lima | Aislar Finanzas (Vlan113 out) | Anti-escaneo servidores (Vlan118 out) |
| Libertad | Segmentar WiFi-Clientes (Vlan205 in) | FTP solo Admin (Vlan207 out) |
| Ica | Proteger gestión 999 (Vlan999 out) | DNS solo local (G1/0/1 out) |
| Huánuco | Aislar Logística (Vlan402 in) | Solo-Web WiFi-Ejec (Vlan406 in) |
| Puno | Ventas → servicios centrales (Vlan503 in) | Bloquear ICMP a servidores (Vlan507 out) |

### Criterio de colocación (principio de ingeniería)
- **ACL estándar** → cerca del destino (filtra por IP origen).
- **ACL extendida** → cerca del origen (filtra origen + destino + puerto).
- **Nunca** aplicar filtrado en interfaces con crypto map (VPN).

---

## 2. POLÍTICA EMPRESARIAL / INTERSEDE

**Dispositivo:** ROUTER_LIMA
**Objetivo:** Hub-and-spoke seguro. Las sucursales hablan con Lima e Internet,
pero NO entre sí. Aislamiento inter-sucursal.

```cisco
ip access-list extended ACL-INTERSEDE
 permit ip 10.192.36.0 0.0.3.255 any
 permit ip 10.192.44.0 0.0.3.255 10.192.40.0 0.0.3.255
 permit ip 10.192.48.0 0.0.3.255 10.192.40.0 0.0.3.255
 permit ip 10.192.52.0 0.0.3.255 10.192.40.0 0.0.3.255
 permit ip 10.192.56.0 0.0.3.255 10.192.40.0 0.0.3.255
 permit ip 10.192.44.0 0.0.3.255 200.100.50.0 0.0.0.255
 permit ip 10.192.48.0 0.0.3.255 200.100.50.0 0.0.0.255
 permit ip 10.192.52.0 0.0.3.255 200.100.50.0 0.0.0.255
 permit ip 10.192.56.0 0.0.3.255 200.100.50.0 0.0.0.255
 deny   ip 10.192.44.0 0.0.15.255 10.192.44.0 0.0.15.255
 permit ip 10.192.44.0 0.0.15.255 any
!
interface Serial0/0/0
 ip access-group ACL-INTERSEDE in
interface Serial0/3/1
 ip access-group ACL-INTERSEDE in
interface Serial0/2/1
 ip access-group ACL-INTERSEDE in
interface Serial0/1/1
 ip access-group ACL-INTERSEDE in
```

**Nota clave:** la 1ra línea (`10.192.36.0/22 → any`) protege el tráfico RIP.
Sin ella, el deny implícito corta los updates y la red dinámica colapsa.
Las 4 sucursales se listan por /22 individual porque el bloque 44-59 cruza
la frontera de alineación /20 y no admite resumen en una sola wildcard.

**Persistencia:** SÍ (vive en router).

---

## 3. LIMA

### 3.1 Política IP — Aislar Finanzas
**CORE_LIMA · Vlan113 (Finanzas) · out**
Solo VLAN de Administración gestiona Finanzas (dato sensible).

```cisco
ip access-list standard ACL-STD-FIN-LIMA
 permit 10.192.41.0 0.0.0.127
 permit 10.192.42.160 0.0.0.31
 permit 10.192.41.128 0.0.0.63
 permit 200.100.50.0 0.0.0.255
 deny   any
!
interface Vlan113
 ip access-group ACL-STD-FIN-LIMA out
```

### 3.2 Política de Servicios — Anti-escaneo de servidores
**CORE_LIMA · Vlan118 (Servidores) · out**
El segmento de servidores solo expone sus servicios legítimos;
cualquier otro puerto se bloquea (anti-escaneo/anti-uso indebido).

```cisco
ip access-list extended ACL-SRV-LIMA
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit udp any host 10.192.42.163 eq domain
 permit tcp any host 10.192.42.163 eq domain
 permit tcp any host 10.192.42.162 eq www
 permit tcp any host 10.192.42.165 eq ftp
 permit tcp any host 10.192.42.165 eq 20
 permit tcp any host 10.192.42.166 eq smtp
 permit tcp any host 10.192.42.166 eq pop3
 permit icmp any 10.192.42.160 0.0.0.31
 deny   ip any 10.192.42.160 0.0.0.31
 permit ip any any
!
interface Vlan118
 ip access-group ACL-SRV-LIMA out
```

---

## 4. LIBERTAD

### 4.1 Política IP — Segmentar WiFi-Clientes
**CORE_LIBERTAD · Vlan205 (WiFi-Cli) · in**
Los invitados WiFi solo acceden a Internet + DNS local; nada del corporativo.

```cisco
ip access-list extended ACL-IP-WIFICLI-LIBERTAD
 permit udp any any eq bootps
 permit udp 10.192.45.128 0.0.0.63 host 10.192.46.3 eq domain
 permit tcp 10.192.45.128 0.0.0.63 host 10.192.46.3 eq domain
 deny   ip  10.192.45.128 0.0.0.63 10.0.0.0 0.255.255.255
 permit ip  10.192.45.128 0.0.0.63 any
!
interface Vlan205
 ip access-group ACL-IP-WIFICLI-LIBERTAD in
```

### 4.2 Política de Servicios — FTP solo Admin
**CORE_LIBERTAD · Vlan207 (Servidores) · out**
Solo Administración usa FTP; el resto de servicios (DNS/Web) abiertos a todos.

```cisco
ip access-list extended ACL-SRV-LIBERTAD
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit udp any host 10.192.46.3 eq domain
 permit tcp any host 10.192.46.3 eq domain
 permit tcp any host 10.192.46.2 eq www
 permit tcp 10.192.44.0 0.0.0.127 host 10.192.46.5 eq ftp
 permit tcp 10.192.44.0 0.0.0.127 host 10.192.46.5 eq 20
 deny   tcp any host 10.192.46.5 eq ftp
 deny   tcp any host 10.192.46.5 eq 20
 permit icmp any 10.192.46.0 0.0.0.31
 permit ip any any
!
interface Vlan207
 ip access-group ACL-SRV-LIBERTAD out
```

---

## 5. ICA

### 5.1 Política IP — Proteger gestión
**CORE_ICA · Vlan999 (Gestión) · out**
Solo la VLAN de gestión alcanza la subred de administración; ningún usuario.
Única defensa del plano de control (VTY-MGMT no aplicada en esta versión).

```cisco
ip access-list extended ACL-IP-MGMT-ICA
 permit ip 10.192.60.96 0.0.0.31 10.192.60.96 0.0.0.31
 deny   ip any 10.192.60.96 0.0.0.31
 permit ip any any
!
interface Vlan999
 ip access-group ACL-IP-MGMT-ICA out
```

### 5.2 Política de Servicios — DNS solo local (anti-rogue)
**CORE_ICA · GigabitEthernet1/0/1 (salida al router) · out**
Los usuarios solo resuelven contra el DNS local; DNS externo bloqueado
(previene DNS rogue y DNS tunneling).

```cisco
ip access-list extended ACL-SRV-DNS-ICA
 permit udp any host 10.192.50.195 eq domain
 permit tcp any host 10.192.50.195 eq domain
 deny   udp any any eq domain
 deny   tcp any any eq domain
 permit ip any any
!
interface GigabitEthernet1/0/1
 ip access-group ACL-SRV-DNS-ICA out
```

---

## 6. HUÁNUCO

### 6.1 Política IP — Aislar Logística
**CORE_HUANUCO · Vlan402 (Logística) · in**
Logística solo habla con servidores locales, Mail Lima e Internet.
Contención de área operativa.

```cisco
ip access-list extended ACL-IP-LOG-HUANUCO
 permit udp any any eq bootps
 permit ip 10.192.53.0 0.0.0.127 10.192.53.0 0.0.0.127
 permit ip 10.192.53.0 0.0.0.127 10.192.54.128 0.0.0.31
 permit ip 10.192.53.0 0.0.0.127 host 10.192.42.166
 deny   ip 10.192.53.0 0.0.0.127 10.192.0.0 0.0.255.255
 permit ip 10.192.53.0 0.0.0.127 any
!
interface Vlan402
 ip access-group ACL-IP-LOG-HUANUCO in
```

### 6.2 Política de Servicios — Solo-Web
**CORE_HUANUCO · Vlan406 (WiFi-Ejecutivos) · in**
La zona solo hace navegación web (HTTP/HTTPS) + DNS local; el resto bloqueado.

```cisco
ip access-list extended ACL-SRV-WEB-HUANUCO
 permit udp any any eq bootps
 permit udp 10.192.54.64 0.0.0.63 host 10.192.54.131 eq domain
 permit tcp 10.192.54.64 0.0.0.63 host 10.192.54.131 eq domain
 permit tcp 10.192.54.64 0.0.0.63 any eq www
 permit tcp 10.192.54.64 0.0.0.63 any eq 443
 deny   ip  10.192.54.64 0.0.0.63 any
!
interface Vlan406
 ip access-group ACL-SRV-WEB-HUANUCO in
```

---

## 7. PUNO

### 7.1 Política IP — Ventas solo a servicios centrales
**CORE_PUNO · Vlan503 (Ventas) · in**
Ventas alcanza servidores locales + servidores centrales de Lima + Internet;
bloqueada hacia otras VLANs de usuario (sin movimiento lateral).

```cisco
ip access-list extended ACL-IP-VENT-PUNO
 permit udp any any eq bootps
 permit ip 10.192.57.128 0.0.0.63 10.192.57.128 0.0.0.63
 permit ip 10.192.57.128 0.0.0.63 10.192.58.96 0.0.0.31
 permit ip 10.192.57.128 0.0.0.63 10.192.42.160 0.0.0.31
 deny   ip 10.192.57.128 0.0.0.63 10.192.0.0 0.0.255.255
 permit ip 10.192.57.128 0.0.0.63 any
!
interface Vlan503
 ip access-group ACL-IP-VENT-PUNO in
```

### 7.2 Política de Servicios — Bloquear ICMP a servidores
**CORE_PUNO · Vlan507 (Servidores) · out**
Los servicios responden (DNS/Web/FTP/DHCP), pero el segmento no es
pingeable (anti-reconocimiento).

```cisco
ip access-list extended ACL-SRV-ICMP-PUNO
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit udp any host 10.192.58.99 eq domain
 permit tcp any host 10.192.58.99 eq domain
 permit tcp any host 10.192.58.98 eq www
 permit tcp any host 10.192.58.101 eq ftp
 permit tcp any host 10.192.58.101 eq 20
 deny   icmp any 10.192.58.96 0.0.0.31 echo
 permit ip any any
!
interface Vlan507
 ip access-group ACL-SRV-ICMP-PUNO out
```

**Fallback:** si `echo` da error de sintaxis, reemplazar esa línea por
`deny icmp any 10.192.58.96 0.0.0.31` (bloquea todo ICMP a servidores).

---

## 8. NOTAS DE ACTIVACIÓN (IMPORTANTE)

### Bug de persistencia — Packet Tracer 9
Las ACLs aplicadas en **SVIs de switches multicapa 3650** no conservan el
binding `ip access-group` al guardar/reabrir el archivo .pkt. Las
**definiciones** de las ACLs SÍ persisten; solo el binding a la interfaz
se pierde.

**Procedimiento tras cada apertura del .pkt:** reaplicar los bindings.
Las ACLs no se recrean (ya están guardadas).

Excepción: la política Intersede (ROUTER_LIMA) y la de servicios de Ica
(puerto ruteado G1/0/1) tienen mayor probabilidad de persistir por no
vivir en SVIs.

### Script de reactivación de bindings

```cisco
! ===== CORE_LIMA =====
interface Vlan113
 ip access-group ACL-STD-FIN-LIMA out
interface Vlan118
 ip access-group ACL-SRV-LIMA out
!
! ===== CORE_LIBERTAD =====
interface Vlan205
 ip access-group ACL-IP-WIFICLI-LIBERTAD in
interface Vlan207
 ip access-group ACL-SRV-LIBERTAD out
!
! ===== CORE_ICA =====
interface Vlan999
 ip access-group ACL-IP-MGMT-ICA out
interface GigabitEthernet1/0/1
 ip access-group ACL-SRV-DNS-ICA out
!
! ===== CORE_HUANUCO =====
interface Vlan402
 ip access-group ACL-IP-LOG-HUANUCO in
interface Vlan406
 ip access-group ACL-SRV-WEB-HUANUCO in
!
! ===== CORE_PUNO =====
interface Vlan503
 ip access-group ACL-IP-VENT-PUNO in
interface Vlan507
 ip access-group ACL-SRV-ICMP-PUNO out
```

---

## 9. VERIFICACIÓN FUNCIONAL

Usar pings/servicios en modo Realtime (no confiar en
`show ip interface Vlan | include access list`, que reporta `not set`
erróneamente en 3650 de PT aunque el binding esté activo).

| Prueba | Resultado esperado |
|--------|-------------------|
| Ventas Lima → Finanzas Lima | FALLA |
| WiFi-Cli Libertad → corporativo | FALLA / Internet OK |
| FTP Admin Libertad → servidor | OK / FTP otros → FALLA |
| Usuario Ica → subred gestión 999 | FALLA |
| Ica → DNS externo (8.8.8.8) | FALLA / DNS local OK |
| Logística HCO → otras VLANs | FALLA / servidores OK |
| WiFi-Ejec HCO → web | OK / otros puertos FALLA |
| Ventas Puno → otras VLANs | FALLA / Lima central OK |
| ping a servidores Puno | FALLA / DNS-Web OK |

---

## 10. SÍNTESIS DE VECTORES DE SEGURIDAD

Las 10 políticas sede-específicas cubren vectores distintos:

**Políticas IP (capa 3):**
1. Aislar destino sensible (Lima - Finanzas)
2. Segmentar zona no confiable (Libertad - WiFi)
3. Proteger plano de control (Ica - Gestión)
4. Contener área operativa (Huánuco - Logística)
5. Restringir a servicios centrales (Puno - Ventas)

**Políticas de Servicios (capa 4):**
1. Anti-escaneo por puertos (Lima)
2. Restricción de servicio por rol (Libertad - FTP/Admin)
3. Anti-DNS-rogue / tunneling (Ica)
4. Navegación controlada (Huánuco - Solo-Web)
5. Anti-reconocimiento ICMP (Puno)
