# INFORME — PROTOCOLOS PUNTO A PUNTO (PPP)
## Proyecto MIEMPRESA 

---

## 1. RESUMEN EJECUTIVO

Todos los enlaces WAN seriales de la red MIEMPRESA usan **PPP
(Point-to-Point Protocol)** con autenticación. Se aplican dos métodos
según la zona de la red:

- **CHAP** en los enlaces de la zona pública (ROUTER_LIMA ↔ ISPs, ISP1 ↔ ISP2).
- **PAP** en todos los enlaces de la red interna corporativa
  (sucursales de Perú + los 4 routers internacionales).

### Criterio de diseño
CHAP en la frontera pública (mayor seguridad: no envía la contraseña en
claro, usa desafío-respuesta con hash); PAP en la red interna (más simple,
suficiente para un entorno controlado).

---

## 2. MAPA DE AUTENTICACIÓN PPP

| Enlace | Autenticación | Zona |
|--------|---------------|------|
| Lima Se0/1/0 ↔ ISP1 | CHAP | Pública |
| Lima Se0/2/0 ↔ ISP2 | CHAP | Pública |
| ISP1 ↔ ISP2 (Se0/2/1) | CHAP | Pública |
| Lima ↔ Libertad | PAP | Interna |
| Lima ↔ Ica | PAP | Interna |
| Lima ↔ Huánuco | PAP | Interna |
| Lima ↔ Puno | PAP | Interna |
| Lima ↔ Quito | PAP | Interna (intl) |
| Lima ↔ SCH | PAP | Interna (intl) |
| Quito ↔ Bogotá | PAP | Interna (intl) |
| SCH ↔ BA | PAP | Interna (intl) |

Credencial de laboratorio: `cisco123` (uniforme en todos los enlaces).

---

## 3. FUNDAMENTOS PPP

### 3.1 Componentes
- **LCP (Link Control Protocol):** establece, configura y prueba el enlace.
- **NCP (Network Control Protocol):** negocia protocolos de capa 3 (IPCP
  para IP).
- **Autenticación:** fase opcional entre LCP y NCP (PAP o CHAP).

### 3.2 Encapsulación
```cisco
interface SerialX/X/X
 encapsulation ppp
```
Sin este comando, la interfaz usa HDLC (default Cisco), que no soporta
autenticación.

---

## 4. AUTENTICACIÓN CHAP (Zona Pública)

### 4.1 Características
- **Challenge-Handshake Authentication Protocol.**
- Usa un secreto **compartido idéntico** en ambos extremos.
- No envía la contraseña por el enlace: usa un desafío (challenge) y
  responde con un hash MD5.
- Autenticación de tres vías, se repite periódicamente durante la sesión.
- `username = hostname del vecino`.

### 4.2 Configuración — ROUTER_LIMA (lado ISP)

```cisco
username ISP1 password cisco123
username ISP2 password cisco123
!
interface Serial0/1/0
 ppp authentication chap
!
interface Serial0/2/0
 ppp authentication chap
```

### 4.3 Configuración — ISP1

```cisco
username ROUTER_LIMA password cisco123
username ISP2 password cisco123
!
interface Serial0/2/0
 ppp authentication chap
!
interface Serial0/2/1
 ppp authentication chap
```

### 4.4 Configuración — ISP2

```cisco
username ROUTER_LIMA password cisco123
username ISP1 password cisco123
!
interface Serial0/2/0
 ppp authentication chap
!
interface Serial0/2/1
 ppp authentication chap
```

---

## 5. AUTENTICACIÓN PAP (Red Interna)

### 5.1 Características
- **Password Authentication Protocol.**
- Autenticación de dos vías: cada router envía su usuario/contraseña.
- Más simple que CHAP (menor overhead), adecuado para red interna
  controlada.
- Cada router envía su propio hostname como `sent-username` y valida al
  vecino con el `username` global.

### 5.2 Patrón de configuración PAP

```cisco
! En cada router:
username <HOSTNAME-VECINO> password cisco123
!
interface SerialX/X/X
 ppp authentication pap
 ppp pap sent-username <HOSTNAME-PROPIO> password cisco123
```

### 5.3 Ejemplo — ROUTER_LIMA (lado interno)

```cisco
username ROUTER_ICA password cisco123
username ROUTER_PUNO password cisco123
username ROUTER_QUITO password cisco123
username ROUTER_SCH password cisco123
!
interface Serial0/3/1
 ppp authentication pap
 ppp pap sent-username ROUTER_LIMA password cisco123
!
interface Serial0/1/1
 ppp authentication pap
 ppp pap sent-username ROUTER_LIMA password cisco123
!
interface Serial0/0/1
 ppp authentication pap
 ppp pap sent-username ROUTER_LIMA password cisco123
!
interface Serial0/3/0
 ppp authentication pap
 ppp pap sent-username ROUTER_LIMA password cisco123
```

### 5.4 Ejemplo — ROUTER_ICA (extremo sucursal)

```cisco
username ROUTER_LIMA password cisco123
!
interface Serial0/1/0
 ppp authentication pap
 ppp pap sent-username ROUTER_ICA password cisco123
```

### 5.5 Ejemplo — ROUTER_QUITO (router de tránsito, 2 enlaces PAP)

```cisco
username ROUTER_LIMA password cisco123
username ROUTER_BOGOTA password cisco123
!
interface Serial0/1/0
 ppp authentication pap
 ppp pap sent-username ROUTER_QUITO password cisco123
!
interface Serial0/2/0
 ppp authentication pap
 ppp pap sent-username ROUTER_QUITO password cisco123
```

---

## 6. ASIGNACIÓN DCE/DTE (Clock Rate)

En enlaces seriales de Packet Tracer, el lado **DCE** provee la señal de
reloj. Regla aplicada: el extremo "más cercano a Lima" es DCE.

```cisco
! En el lado DCE:
interface SerialX/X/X
 clock rate 64000
```

Ejemplos:
- Lima Se0/1/0 (hacia ISP1) → clock rate en el lado ISP (DCE)
- Lima Se0/0/1 (hacia Quito) → Lima es DCE
- Quito Se0/2/0 (hacia Bogotá) → Quito es DCE

Verificación del lado del cable:
```cisco
show controllers Serial0/1/0
! indica "DCE cable" o "DTE cable"
```

---

## 7. DIFERENCIAS CHAP vs PAP

| Aspecto | PAP | CHAP |
|---------|-----|------|
| Vías | 2 (bidireccional) | 3 (challenge-response) |
| Contraseña en el enlace | En claro | Nunca (hash MD5) |
| Secreto | Por dirección (sent-username) | Compartido idéntico |
| Repetición | Solo al inicio | Periódica durante sesión |
| Seguridad | Básica | Alta |
| Uso en el proyecto | Red interna | Frontera pública (ISPs) |

---

## 8. BUGS Y SOLUCIONES (durante implementación)

| # | Problema | Solución |
|---|----------|----------|
| 1 | PPP-PAP en estado "fantasma" (LCP Open pero sin tráfico) tras cambiar hostnames/autenticación | Limpiar username viejos, reaplicar sent-username, shutdown/no shutdown coordinado |
| 2 | Enlace no levanta pese a ambos lados configurados | shutdown/no shutdown desde el lado Lima |
| 3 | CHAP falla con secretos distintos | Verificar password idéntica en ambos extremos |
| 4 | Rebote masivo de enlaces tira la ruta default en COREs | Migrar un enlace a la vez; reafirmar default-information originate |

### Precaución operativa clave
Cambiar la autenticación PPP **rebota el enlace** (LCP renegocia). Rebotar
varios enlaces simultáneamente hace reconverger RIP y puede expirar la ruta
default en los COREs. **Regla:** migrar de a un enlace por vez, verificando
convergencia entre cada cambio.

---

## 9. VERIFICACIÓN

### Estado del enlace
```cisco
show ip interface brief | include Serial
! los seriales deben estar up/up

show interfaces Serial0/1/0
! line protocol is up, encapsulation PPP
```

### Estado de la autenticación PPP
```cisco
show ppp interface Serial0/1/0
! LCP Open, IPCP Open, autenticación (CHAP/PAP) exitosa
```

### Prueba de conectividad
```cisco
! Ping al vecino directo del enlace:
ping <IP-vecino-serial>
! debe responder 100% si PPP + autenticación están OK
```

### Interacción con VPN
Los enlaces Lima↔Quito y Lima↔SCH llevaban crypto map (VPN cascada, luego
descartada). El cambio de autenticación PPP NO afecta IPSec (capas
distintas), pero rebota el enlace y las SAs renegocian.

---

## 10. SÍNTESIS

| Zona | Enlaces | Protocolo | Autenticación |
|------|---------|-----------|---------------|
| Pública (ISPs) | Lima↔ISP1, Lima↔ISP2, ISP1↔ISP2 | PPP | CHAP |
| Interna Perú | Lima↔4 sucursales | PPP | PAP |
| Internacional | Lima↔Quito, Lima↔SCH, Quito↔Bogotá, SCH↔BA | PPP | PAP |

**Diseño:** PPP con encapsulación en todos los enlaces WAN. CHAP protege
la frontera pública (desafío-respuesta, sin contraseña en claro); PAP en
la red interna (simplicidad en entorno controlado). Todos los enlaces
verificados up/up con autenticación exitosa y RIP convergiendo sobre ellos.