# INFORME — SPANNING TREE Y BALANCEO DE CARGA
## Proyecto MIEMPRESA

---

## 1. RESUMEN EJECUTIVO

La red MIEMPRESA implementa **Rapid-PVST** (Rapid Per-VLAN Spanning Tree)
en las 5 sedes, con **balanceo de carga por VLAN** sobre los enlaces
redundantes de cada sede.

- **Prevención de bucles:** STP bloquea enlaces redundantes para evitar
  loops en la topología de anillo.
- **Balanceo de carga:** distintos grupos de VLANs usan distintos enlaces
  del anillo, aprovechando ambos caminos en lugar de dejar uno ocioso.
- **Redundancia:** ante la caída de un enlace, STP reactiva el bloqueado
  (failover automático).

---

## 2. TOPOLOGÍA STP POR SEDE

Cada sede tiene una topología en **anillo (triángulo)** entre el CORE y
dos switches de distribución multicapa (D2 y D3):

```
              [ CORE_SEDE ]  ← Root Bridge
              Gi1/0/3   Gi1/0/4
                 |         |
              Gi1/0/1   Gi1/0/1
            [ D2_SEDE ]—[ D3_SEDE ]
              Gi1/0/2   Gi1/0/2
                 └── enlace lateral ──┘
```

Tres enlaces forman el anillo:
- CORE ↔ D2 (uplink directo de D2)
- CORE ↔ D3 (uplink directo de D3)
- D2 ↔ D3 (enlace lateral)

STP bloquea uno de los tres para romper el lazo. El switch de acceso D1
(que gestiona WiFi) tiene un solo uplink al CORE — sin redundancia, no
participa del balanceo.

---

## 3. CONFIGURACIÓN BASE DE STP

### 3.1 Modo Rapid-PVST (todos los switches)

```cisco
spanning-tree mode rapid-pvst
```

Rapid-PVST provee convergencia rápida (segundos vs los ~50s del STP
clásico) y una instancia de árbol por VLAN (necesario para el balanceo).

### 3.2 CORE como Root Bridge (en cada CORE)

```cisco
spanning-tree vlan 1,111-118,999 priority 24576
```

La prioridad 24576 (menor que el default 32768) convierte al CORE en el
**root bridge** de todas las VLANs de su sede. El root nunca bloquea sus
puertos; el bloqueo ocurre en D2 o D3.

---

## 4. BALANCEO DE CARGA POR VLAN

### 4.1 Estrategia

El balanceo reparte las VLANs de usuario en dos grupos:

- **Grupo A** → usa el **uplink directo** al CORE (camino corto, 1 salto).
- **Grupo B** → usa el **enlace lateral** hacia el otro switch de
  distribución (camino de 2 saltos).

Las **VLANs de servidores** se asignan al Grupo A (camino directo) para
garantizar la menor latencia posible a los servicios críticos.
Las **VLANs WiFi** (en D1) quedan fuera del balanceo por no tener
redundancia física.

### 4.2 Método — Costo de puerto por VLAN

El balanceo se configura ajustando el **costo STP por VLAN** en ambos
puertos (uplink Gi1/0/1 y lateral Gi1/0/2) de D2 y D3:

- Grupo A: costo bajo (4) en el uplink, alto (100) en el lateral.
- Grupo B: costo alto (100) en el uplink, bajo (4) en el lateral.

Al tener costos invertidos por grupo, el árbol de spanning-tree converge
distinto para cada grupo de VLANs, repartiendo el tráfico.

### 4.3 Asignación de grupos por sede

| Sede | Grupo A (directo): usuarios + servidores | Grupo B (lateral) |
|------|------------------------------------------|-------------------|
| Lima | 111, 112, 113, 118(srv) | 115, 116 |
| Libertad | 201, 202, 203, 207(srv) | 204, 208 |
| Ica | 301, 302, 303, 307(srv) | 304, 308 |
| Huánuco | 401, 402, 403, 407(srv) | 404, 408 |
| Puno | 501, 502, 503, 507(srv) | 504, 508 |

---

## 5. SCRIPTS DE BALANCEO POR SEDE

Se aplican en **D2 y D3** de cada sede (ambos, mismo script).
Se usan VLANs listadas explícitamente (no rangos) para evitar el bug de
serialización de Packet Tracer.

### LIMA — D2_LIMA y D3_LIMA

```cisco
interface GigabitEthernet1/0/1
 spanning-tree vlan 111,112,113,118 cost 4
 spanning-tree vlan 115,116 cost 100
interface GigabitEthernet1/0/2
 spanning-tree vlan 111,112,113,118 cost 100
 spanning-tree vlan 115,116 cost 4
```

### LIBERTAD — D2_LIBERTAD y D3_LIBERTAD

```cisco
interface GigabitEthernet1/0/1
 spanning-tree vlan 201,202,203,207 cost 4
 spanning-tree vlan 204,208 cost 100
interface GigabitEthernet1/0/2
 spanning-tree vlan 201,202,203,207 cost 100
 spanning-tree vlan 204,208 cost 4
```

### ICA — D2_ICA y D3_ICA

```cisco
interface GigabitEthernet1/0/1
 spanning-tree vlan 301,302,303,307 cost 4
 spanning-tree vlan 304,308 cost 100
interface GigabitEthernet1/0/2
 spanning-tree vlan 301,302,303,307 cost 100
 spanning-tree vlan 304,308 cost 4
```

### HUÁNUCO — D2_HUANUCO y D3_HUANUCO

```cisco
interface GigabitEthernet1/0/1
 spanning-tree vlan 401,402,403,407 cost 4
 spanning-tree vlan 404,408 cost 100
interface GigabitEthernet1/0/2
 spanning-tree vlan 401,402,403,407 cost 100
 spanning-tree vlan 404,408 cost 4
```

### PUNO — D2_PUNO y D3_PUNO

```cisco
interface GigabitEthernet1/0/1
 spanning-tree vlan 501,502,503,507 cost 4
 spanning-tree vlan 504,508 cost 100
interface GigabitEthernet1/0/2
 spanning-tree vlan 501,502,503,507 cost 100
 spanning-tree vlan 504,508 cost 4
```

---

## 6. VERIFICACIÓN

### 6.1 Identificar el switch que bloquea

En cada sede, el bloqueo ocurre en D2 o D3 (depende de las MAC/Bridge ID).
Para identificarlo rápido:

```cisco
show spanning-tree | include BLK
```

El switch que liste puertos en `BLK` es donde se verifica la inversión.

### 6.2 Confirmar la inversión de roles

En el switch que bloquea, comparar una VLAN de cada grupo:

```cisco
show spanning-tree vlan <GRUPO-A>    ! ej. 301
show spanning-tree vlan <GRUPO-B>    ! ej. 304
```

Resultado esperado:

| VLAN | Gi1/0/1 (uplink) | Gi1/0/2 (lateral) | Root Cost |
|------|------------------|-------------------|-----------|
| Grupo A | Root FWD | Altn BLK | 4 (directo) |
| Grupo B | Altn BLK | Root FWD | 8 (2 saltos) |

Si Grupo A usa el uplink directo y Grupo B el lateral, el balanceo está
activo.

### 6.3 Resultados validados (las 5 sedes)

| Sede | Switch que bloquea | VLANs verificadas | Estado |
|------|-------------------|-------------------|--------|
| Lima | D2 | 111 (A) / 115 (B) | Balanceo OK |
| Libertad | D3 | 201 (A) / 204 (B) | Balanceo OK |
| Ica | D3 | 301 (A) / 304 (B) | Balanceo OK |
| Huánuco | D3 | 401 (A) / 404 (B) | Balanceo OK |
| Puno | D3 | 501 (A) / 504 (B) | Balanceo OK |

Nota: el switch que bloquea varía por sede según las direcciones MAC.
En Lima bloquea D2; en las sucursales bloquea D3. En todos los casos la
inversión de roles se confirma en el switch que bloquea.

---

## 7. GUÍA — CÓMO AGREGAR UNA NUEVA VLAN (con switch 2960 de acceso)

Al agregar una VLAN hay que propagarla por toda la cadena de switches:
**CORE → D1/D2/D3 (distribución) → 2960 (acceso) → host**. Si se omite
algún eslabón, la VLAN queda incompleta (sin ruteo, sin trunk, o sin
balanceo).

### 7.1 Arquitectura de conexión

```
        [ CORE ]  (ruteo inter-VLAN + root STP)
        /   |   \
     [D1] [D2]—[D3]   (distribución, anillo con balanceo)
              \  /
           [ SW-2960 ]  (acceso, dedicado por unidad)
                |
             [ Host PC ]
```

- Los **2960** son switches de acceso dedicados por unidad, conectados por
  trunk a D2 y D3 (dual-homing para redundancia).
- Los hosts se conectan a los **puertos de acceso del 2960**.
- **D1** gestiona los AP WiFi; los 2960 de datos cuelgan de D2/D3.

### 7.2 Regla de asignación de grupo de balanceo

- VLAN de usuarios de número bajo → **Grupo A** (uplink directo).
- VLAN de usuarios de número alto → **Grupo B** (enlace lateral).
- VLAN de servidores → **Grupo A** (baja latencia).
- VLAN WiFi → en D1, **fuera del balanceo**.

---

### 7.3 Agregar VLAN en LIMA (ejemplo: VLAN 119, Grupo B)

**Paso 1 — Crear la VLAN en TODOS los switches de la cadena:**
```cisco
! En CORE_LIMA, D1_LIMA, D2_LIMA, D3_LIMA y el nuevo SW-2960
configure terminal
vlan 119
 name NUEVA-UNIDAD-LIMA
 exit
```

**Paso 2 — Crear la SVI (ruteo) en el CORE:**
```cisco
! Solo en CORE_LIMA
interface Vlan119
 ip address <gateway> <mascara>
 ip helper-address 10.192.42.164    ! DHCP Lima
 exit
```

**Paso 3 — Permitir la VLAN en los trunks del anillo (CORE-D2-D3):**
```cisco
! En CORE_LIMA (puertos hacia D2 y D3)
interface range GigabitEthernet1/0/3 - 4
 switchport trunk allowed vlan add 119
 exit
!
! En D2_LIMA y D3_LIMA (uplink Gi1/0/1 + lateral Gi1/0/2)
interface range GigabitEthernet1/0/1 - 2
 switchport trunk allowed vlan add 119
 exit
```

**Paso 4 — Permitir la VLAN en el trunk hacia el 2960:**
```cisco
! En D2_LIMA y D3_LIMA (el puerto que va al nuevo 2960, ej. Gi1/0/13)
interface GigabitEthernet1/0/13
 switchport trunk allowed vlan add 119,999
 exit
```

**Paso 5 — Configurar el trunk del lado del 2960:**
```cisco
! En el nuevo SW-2960 (puerto uplink hacia D2/D3, ej. Gi0/1 y Gi0/2)
interface range GigabitEthernet0/1 - 2
 switchport trunk native vlan 999
 switchport trunk allowed vlan 119,999
 switchport mode trunk
 exit
```

**Paso 6 — Asignar la VLAN a su grupo de balanceo (D2 y D3):**
```cisco
! En D2_LIMA y D3_LIMA — Grupo B (lateral)
interface GigabitEthernet1/0/1
 spanning-tree vlan 119 cost 100
interface GigabitEthernet1/0/2
 spanning-tree vlan 119 cost 4
```
(Grupo A: invertir → cost 4 en Gi1/0/1, cost 100 en Gi1/0/2.)

**Paso 7 — Prioridad de root del CORE:**
```cisco
! En CORE_LIMA
spanning-tree vlan 119 priority 24576
```

**Paso 8 — Puertos de acceso del host en el 2960:**
```cisco
! En el SW-2960
interface range FastEthernet0/1 - 2
 switchport access vlan 119
 switchport mode access
 spanning-tree portfast
 exit
```

**Paso 9 — Guardar y verificar:**
```cisco
! En cada switch tocado
end
write memory
!
! Verificar
show spanning-tree vlan 119        ! balanceo activo (en el switch que bloquea)
show vlan brief | include 119      ! VLAN activa
! Conectar un PC al 2960 → debe recibir IP por DHCP
```

---

### 7.4 Agregar VLAN en SUCURSAL (ejemplo: VLAN 209 en Libertad, Grupo A)

Procedimiento idéntico, ajustando numeración, DHCP helper y grupo.

**Paso 1 — Crear la VLAN (CORE, D1, D2, D3, 2960 de la sucursal):**
```cisco
configure terminal
vlan 209
 name NUEVA-UNIDAD-LIBERTAD
 exit
```

**Paso 2 — SVI en el CORE de la sucursal:**
```cisco
! En CORE_LIBERTAD
interface Vlan209
 ip address <gateway> <mascara>
 ip helper-address 10.192.46.4      ! DHCP Libertad (local)
 exit
```

**Paso 3 — Trunks del anillo:**
```cisco
! En CORE_LIBERTAD (hacia D2 y D3)
interface range GigabitEthernet1/0/3 - 4
 switchport trunk allowed vlan add 209
!
! En D2_LIBERTAD y D3_LIBERTAD
interface range GigabitEthernet1/0/1 - 2
 switchport trunk allowed vlan add 209
```

**Paso 4 — Trunk hacia el 2960 (lado D2/D3):**
```cisco
! En D2_LIBERTAD y D3_LIBERTAD (puerto al nuevo 2960)
interface GigabitEthernet1/0/13
 switchport trunk allowed vlan add 209,999
```

**Paso 5 — Trunk del lado 2960:**
```cisco
! En el nuevo SW-2960 de Libertad
interface range GigabitEthernet0/1 - 2
 switchport trunk native vlan 999
 switchport trunk allowed vlan 209,999
 switchport mode trunk
```

**Paso 6 — Grupo de balanceo (D2 y D3) — Grupo A (directo):**
```cisco
! En D2_LIBERTAD y D3_LIBERTAD
interface GigabitEthernet1/0/1
 spanning-tree vlan 209 cost 4
interface GigabitEthernet1/0/2
 spanning-tree vlan 209 cost 100
```

**Paso 7 — Root priority en el CORE:**
```cisco
! En CORE_LIBERTAD
spanning-tree vlan 209 priority 24576
```

**Paso 8 — Acceso del host en el 2960:**
```cisco
interface range FastEthernet0/1 - 2
 switchport access vlan 209
 switchport mode access
 spanning-tree portfast
```

**Paso 9 — Guardar y verificar** (igual que Lima).

---

### 7.5 Caso especial — VLAN WiFi en D1 (sin balanceo)

Si la VLAN nueva es de WiFi, cuelga de **D1** (que gestiona los AP) y
**no** lleva balanceo (D1 tiene un solo uplink al CORE).

```cisco
! Crear VLAN en CORE y D1
vlan <N>
 name WIFI-NUEVA
!
! SVI en CORE con DHCP helper
interface Vlan<N>
 ip address <gateway> <mascara>
 ip helper-address <DHCP-sede>
!
! Trunk CORE↔D1 permitir la VLAN
! En CORE (puerto hacia D1) y en D1 (uplink)
 switchport trunk allowed vlan add <N>
!
! Puerto del AP en D1 (modo access)
interface GigabitEthernet1/0/12
 switchport access vlan <N>
 switchport mode access
```
NO se aplican comandos `spanning-tree cost` (D1 no tiene redundancia).

---

### 7.6 Checklist de propagación (para no olvidar eslabones)

| Dispositivo | Qué se configura |
|-------------|------------------|
| CORE | VLAN + SVI + helper + trunk a D2/D3 + root priority |
| D1 | VLAN + trunk (solo si es WiFi) |
| D2 y D3 | VLAN + trunk anillo + trunk al 2960 + costo de balanceo |
| SW-2960 | VLAN + trunk uplink + puertos de acceso del host |

Si falta el **CORE** → sin ruteo/DHCP. Si falta un **trunk** → la VLAN no
pasa. Si falta el **costo en D2/D3** → sin balanceo. Si falta el **2960**
→ el host no tiene puerto de acceso.

### 7.7 Diferencia clave Lima vs Sucursal

| Aspecto | Lima | Sucursal |
|---------|------|----------|
| DHCP helper | 10.192.42.164 | DHCP local de la sede |
| Numeración VLAN | 11X | 2XX/3XX/4XX/5XX |
| Switch que bloquea | D2 | D3 |
| 2960 de acceso | Dual-homed a D2/D3 | Dual-homed a D2/D3 |
| Procedimiento | Idéntico | Idéntico |

## 8. CONSIDERACIONES Y LIMITACIONES

### 8.1 Escalabilidad
Al agregar VLANs, seguir la regla de asignación de grupo (Sección 7.1)
mantiene el balanceo equilibrado. Se recomienda documentar en qué grupo
queda cada VLAN nueva.

### 8.2 Servidores siempre en camino directo
Las VLANs de servidores nunca se asignan al Grupo B, para no introducir
un salto extra en el acceso a servicios críticos (DNS, DHCP, Web, FTP).

### 8.3 Bug de Packet Tracer
En la versión utilizada, los comandos `spanning-tree vlan <rango> cost`
con rangos se corrompen al serializar el archivo (se muestran como
`vlan 1- vlan X`). Solución aplicada: usar VLANs **listadas
explícitamente** (`vlan 111,112,113`) en lugar de rangos. Las líneas
corruptas, cuando aparecen, mantienen su efecto funcional pese a la
visualización errónea.

### 8.4 Redundancia sin balanceo en D1
El switch D1 (WiFi) tiene un solo uplink al CORE. Sus VLANs (WiFi-Cliente
y WiFi-Ejecutivo) no participan del balanceo por carecer de camino
redundante. Es correcto por diseño.

---

## 9. SÍNTESIS

| Elemento | Configuración |
|----------|---------------|
| Modo STP | Rapid-PVST (todas las sedes) |
| Root Bridge | CORE de cada sede (priority 24576) |
| Topología | Anillo CORE-D2-D3 |
| Balanceo | Costo por VLAN en D2 y D3 (Grupo A directo / B lateral) |
| Servidores | Grupo A (camino directo) |
| WiFi (D1) | Fuera del balanceo (single uplink) |
| Resultado | 2 enlaces activos por sede, carga repartida por VLAN |