# INFORME — SERVICIOS DE RED
## Proyecto MIEMPRESA 
## WEB · DNS · DHCP · FTP · MAIL

---

## 1. RESUMEN EJECUTIVO

La red MIEMPRESA implementa cinco servicios distribuidos en dos ámbitos:

- **Red interna (corporativa):** cada sede de Perú tiene servidores locales
  de WEB, DNS, DHCP y FTP. El correo (MAIL) es centralizado en Lima.
- **Red externa (pública):** un sitio público con servidores WEB y DNS
  accesibles bajo el dominio `miempresa.com`.

### Filosofía de diseño
- **Distribución local:** cada sede resuelve DNS, asigna IPs (DHCP) y sirve
  web/FTP localmente → reduce tráfico WAN y da autonomía por sede.
- **Centralización de correo:** un solo MAIL en Lima → gestión unificada,
  correo intra-empresa.
- **Jerarquía DNS:** público → Lima → sucursales.

---

## 2. INVENTARIO DE SERVIDORES

### 2.1 Red interna — servidores por sede

| Sede | WEB | DNS | DHCP | FTP | MAIL |
|------|-----|-----|------|-----|------|
| Lima | 10.192.42.162 | 10.192.42.163 | 10.192.42.164 | 10.192.42.165 | 10.192.42.166 |
| Libertad | 10.192.46.2 | 10.192.46.3 | 10.192.46.4 | 10.192.46.5 | — |
| Ica | 10.192.50.194 | 10.192.50.195 | 10.192.50.196 | 10.192.50.197 | — |
| Huánuco | 10.192.54.130 | 10.192.54.131 | 10.192.54.132 | 10.192.54.133 | — |
| Puno | 10.192.58.98 | 10.192.58.99 | 10.192.58.100 | 10.192.58.101 | — |

Las sucursales NO tienen MAIL (correo centralizado en Lima).

### 2.2 Red externa — sitio público (200.100.50.0/24)

| Servicio | IP | Dominio |
|----------|-----|---------|
| DNS público | 200.100.50.10 | dns.miempresa.com |
| WEB público | 200.100.50.20 | www.miempresa.com |
| Gateway | 200.100.50.1 | (ISP3) |

---

## 3. DHCP

### 3.1 Arquitectura
Cada sede tiene un servidor DHCP local que entrega IPs a **todas las VLANs
de usuarios** de esa sede. Las VLANs de servidores usan IP estática.

### 3.2 IP Helper-Address (en cada CORE)
Como el DHCP está en la VLAN de servidores y los clientes en otras VLANs,
cada SVI de usuario del CORE reenvía las solicitudes al servidor DHCP:

```cisco
! CORE_LIMA (ejemplo)
interface Vlan111
 ip helper-address 10.192.42.164
! ... repetido en todas las SVIs de usuarios
```

| Sede | DHCP Server (helper) |
|------|---------------------|
| Lima | 10.192.42.164 |
| Libertad | 10.192.46.4 |
| Ica | 10.192.50.196 |
| Huánuco | 10.192.54.132 |
| Puno | 10.192.58.100 |

### 3.3 Pools DHCP
Total: **35 pools** (5 sedes × 7 VLANs de usuarios). Cada pool entrega:
- Default Gateway: la SVI de la VLAN
- DNS Server: el DNS local de la sede
- Rango con exclusión de las primeras IPs (gateway + servidores fijos)

Configurados en la GUI de cada servidor (Services → DHCP → On).

---

## 4. DNS

### 4.1 Jerarquía de tres niveles

```
Nivel 1 (top):  DNS PÚBLICO (200.100.50.10) — autoritativo miempresa.com
                        |
Nivel 2:        DNS_LIMA (10.192.42.163) — nivel intermedio
                        |
Nivel 3:        DNS de sucursal (.3/.195/.131/.99) — resolución local
```

### 4.2 Nota técnica de Packet Tracer
PT no soporta forwarders (encadenamiento automático entre DNS). Por eso
cada nivel replica manualmente los registros relevantes. La jerarquía es
de responsabilidad/cobertura, no de delegación dinámica.

### 4.3 Registros tipo A (patrón)

**DNS público:**
```
www.miempresa.com    → 200.100.50.20
dns.miempresa.com    → 200.100.50.10
```

**DNS_LIMA:**
```
www.lima.miempresa.com   → 10.192.42.162
mail.miempresa.com       → 10.192.42.166
www.miempresa.com        → 200.100.50.20
(+ dns/dhcp/ftp locales)
```

**DNS de cada sucursal:**
```
www.<sede>.miempresa.com → WEB local de la sede
mail.miempresa.com       → 10.192.42.166 (MAIL central Lima)
www.miempresa.com        → 200.100.50.20 (Web público)
(+ dns/dhcp/ftp locales)
```

### 4.4 Modelo de nombres

| Nombre | Resuelve a | Disponible desde |
|--------|-----------|------------------|
| www.miempresa.com | 200.100.50.20 (público) | Cualquier sede |
| mail.miempresa.com | 10.192.42.166 (Lima) | Cualquier sede |
| www.lima.miempresa.com | 10.192.42.162 | Registrado en Lima |
| www.<sede>.miempresa.com | WEB local | Su propia sede |

---

## 5. WEB (HTTP)

### 5.1 Servidores web
- **Público:** `www.miempresa.com` → 200.100.50.20 (accesible desde toda
  la red vía NAT/VPN).
- **Local por sede:** `www.<sede>.miempresa.com` → sirve la página interna
  de cada sucursal.

### 5.2 Contenido
Cada servidor web tiene su `index.html` editado con contenido propio de
la sede. Accesible desde el navegador de los PCs (Desktop → Web Browser).

### 5.3 Acceso
```
http://www.miempresa.com          → Web público
http://www.lima.miempresa.com     → Web local Lima
http://www.<sede>.miempresa.com   → Web local de cada sede
```

---

## 6. FTP

### 6.1 Servidores FTP
Uno por sede (Lima .165, Libertad .5, Ica .197, Huánuco .133, Puno .101).

### 6.2 Configuración
- Usuarios configurados en cada servidor (Services → FTP).
- Password de laboratorio: `cisco123`.
- Permisos de lectura/escritura según usuario.

### 6.3 Control de acceso (política de servicios)
En Libertad, el acceso FTP está restringido: **solo la VLAN Administración**
puede usar FTP en el servidor local (política de seguridad de servicios).

---

## 7. MAIL (Correo — SMTP/POP3)

### 7.1 Servidor centralizado
- **MAIL_LIMA:** 10.192.42.166
- Domain: `mail.miempresa.com`
- Servicios: SMTP (envío) + POP3 (recepción), ambos ON.
- Alcance: **intra-empresa únicamente** (no sale a Internet).

### 7.2 Patrón de cuentas
```
pc<N>_<unidad>_<sede>@mail.miempresa.com
```
Ejemplos:
```
pc1_ventas_lima@mail.miempresa.com
pc1_admin_puno@mail.miempresa.com
pc1_fin_libertad@mail.miempresa.com
```
Password de laboratorio: `cisco123`.

### 7.3 Configuración del cliente (en cada PC)
```
Incoming/Outgoing Mail Server: mail.miempresa.com
User Name: pc<N>_<unidad>_<sede>  (sin @dominio)
Password: cisco123
```

Se usa el hostname `mail.miempresa.com` (no la IP) para validar de paso
la resolución DNS desde cualquier sede.

### 7.4 Funcionamiento cross-sede
El correo funciona intra-sede y entre sedes (ej. Puno ↔ Lima), validando
la cadena completa: DNS (resuelve mail.miempresa.com) + routing WAN +
servicio SMTP/POP3.

---

## 8. SERVICIOS EN RED INTERNA vs EXTERNA

| Servicio | Red interna | Red externa (Site) |
|----------|-------------|---------------------|
| DNS | Local por sede + Lima intermedio | Público (autoritativo) |
| WEB | Local por sede | Público (www.miempresa.com) |
| DHCP | Local por sede (5 servidores) | No aplica |
| FTP | Local por sede | No aplica |
| MAIL | Centralizado en Lima | No (solo intra-empresa) |

### Comunicación interna ↔ externa
- Los hosts internos acceden al Web/DNS público vía NAT (salida) y vía
  VPN site-to-site (bidireccional con el segmento 200.100.50.0/24).
- El DNS local de cada sede tiene registrado `www.miempresa.com` apuntando
  al Web público, permitiendo resolución desde cualquier sede.

---

## 9. VERIFICACIÓN

### DHCP
```
! En un PC: IP Configuration → DHCP
! Debe recibir IP, máscara, gateway y DNS del pool correcto
```

### DNS
```
nslookup www.miempresa.com       → 200.100.50.20
nslookup mail.miempresa.com      → 10.192.42.166
nslookup www.<sede>.miempresa.com → WEB local
```

### WEB
```
http://www.miempresa.com         → carga página pública
http://www.<sede>.miempresa.com  → carga página local
```

### MAIL
```
! Test cross-sede (Puno → Lima):
! 1. nslookup mail.miempresa.com  → 10.192.42.166
! 2. Compose + Send desde PC Puno a cuenta de Lima
! 3. Receive en PC Lima → correo llega
! 4. Bidireccional Lima → Puno
```

---

## 10. SÍNTESIS

| Servicio | Ubicación | Alcance | Registro DNS |
|----------|-----------|---------|--------------|
| DHCP | 5 servidores locales | Por sede | dhcp.<sede> |
| DNS | 5 locales + 1 Lima + 1 público | Jerárquico | dns.<sede> |
| WEB | 5 locales + 1 público | Local + global | www / www.<sede> |
| FTP | 5 locales | Por sede | ftp.<sede> |
| MAIL | 1 central (Lima) | Intra-empresa | mail.miempresa.com |

**Diseño:** servicios de infraestructura (DNS/DHCP/Web/FTP) distribuidos
por sede para autonomía y eficiencia WAN; correo centralizado para gestión
unificada; jerarquía DNS de tres niveles conectando lo local con lo público.