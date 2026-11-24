# Práctica — Laboratorio Gestor SNMP (Red 192.168.10.0/24)

**Materia:** Redes 2  
**Docente:** Ing. Gonzales  
**Tipo:** Práctica de repaso, con equipos distintos a los de la entrega (router **R1** y switch **SW1**)  
**Fecha:** 20 de noviembre de 2026  
**Gestor SNMP:** iReasoning MIB Browser (Portátil ASUS)

---

## Estado de avance

| # | Etapa | Estado |
| :-- | :--- | :--- |
| 1 | Auditoría del router R1 | ✅ SNMP ya configurado y guardado |
| 2 | Configuración SNMP del switch SW1 | ✅ Configurado; dos errores de tipeo corregidos |
| 3 | IP de la laptop gestor `192.168.10.10` | ✅ Verificada (se corrigió un APIPA 169.254.x.x) |
| 4 | Conectividad laptop ↔ R1 / SW1 | ✅ Ping 100 % a ambos (el cable de R1 estaba mal conectado) |
| 5 | SNMP GET `sysName.0` a R1 y SW1 | ✅ Responden `R1` y `SW1` |
| 6 | Pruebas en iReasoning | ✅ Identidad, Walk, Flujo, Traps y SET |
| 7 | MIBs de Maipu cargadas y consultadas | ✅ `numBytesFree.0 = 66 523 040` en R1 — **práctica completa** |

---

## 1. Topología y direccionamiento

```
[Laptop Gestor]            [Switch SW1]                    [Router R1]
 192.168.10.10 ───── Ethernet1/1      Ethernet1/2 ───── fastethernet0
                     (ifIndex 1)  Vlan1   (ifIndex 2)     192.168.10.1
                                192.168.10.2
```

| Equipo | Interfaz | IP / Máscara | Gateway | Rol |
| :--- | :--- | :--- | :--- | :--- |
| Router **R1** (Maipu, MyPower 6.1.7) | `fastethernet0` | `192.168.10.1/24` | — | Agente SNMP |
| Switch **SW1** (Maipu SM3000) | `Vlan1` | `192.168.10.2/24` | `192.168.10.1` | Agente SNMP |
| Laptop ASUS | `Ethernet` → SW1 `Ethernet1/1` | `192.168.10.10/24` | `192.168.10.1` | Gestor + Trap Receiver |

> **Diferencia con la entrega:** el laboratorio entregado usaba la red `192.168.1.0/24` (router `router`, switch `switch`). Esta práctica usa `192.168.10.0/24` con equipos llamados `R1` y `SW1`.

---

## 2. Router R1 — auditoría (`show running-config`)

```
hostname R1
!
vlan 1
 description default
 port 0-7 untagged
!
interface fastethernet0
 ip address 192.168.10.1 255.255.255.0
!
interface fastethernet1
!
snmp-server start
snmp-server view default 1.3.6.1 include
snmp-server view readonlyview 1.3.6.1 include
snmp-server view readwriteview 1.3.6.1 include
snmp-server community public view readonlyview ro
snmp-server community private view readwriteview rw
snmp-server host 192.168.10.10 traps community public version 2
snmp-server enable traps snmp linkdown
snmp-server enable traps snmp linkup
```

| Punto | Estado |
| :--- | :--- |
| Agente SNMP activo (`snmp-server start`) | ✅ |
| Lectura `public` → vista `readonlyview` (ro) | ✅ |
| Escritura `private` → vista `readwriteview` (rw) | ✅ Necesaria para el SET |
| Traps hacia `192.168.10.10`, versión 2 | ✅ |
| Traps `linkdown` / `linkup` | ✅ |
| Configuración guardada | ✅ *"No configuration change since last restart"* |

> A diferencia del router de la entrega (una sola vista `default`), R1 usa **vistas separadas** para lectura y escritura. El resultado es el mismo.  
> R1 tiene un módulo switch (`vlan 1`, `port 0-7`) **además** de `fastethernet0`: el cable debe ir al puerto **FE0**, no a los puertos 0–7.

---

## 3. Switch SW1 — configuración SNMP

### 3.1 Estado inicial
`hostname SW1`, `interface Vlan1 192.168.10.2/24`, sin `ip default-gateway` y sin líneas `snmp-server`.

### 3.2 Comandos aplicados (primer intento)
```
config
snmp-server enable
snmp-server securityip disable
snmp-server community ro 0 public
snmp-server community ro 0 private        ← ERROR: debía ser rw
snmp-server host 192.168.10.10 v2c publc  ← ERROR: "publc" sin la i
snmp-server enable traps
ip default-gateway 192.168.10.1
exit
write
```

### 3.3 Errores detectados y corrección

| Error | Consecuencia | Corrección aplicada |
| :--- | :--- | :--- |
| `community ro 0 private` | La comunidad de escritura quedaba de solo lectura: **el SET habría fallado** | `no snmp-server community 0 private` y luego `snmp-server community rw 0 private` |
| `host 192.168.10.10 v2c publc` | Los traps se enviarían con una comunidad errónea | `no snmp-server host 192.168.10.10 v2c publc` y luego `snmp-server host 192.168.10.10 v2c public` |

**Intentos fallidos durante la corrección (y por qué):**

| Comando intentado | Resultado | Motivo |
| :--- | :--- | :--- |
| `no snmp-server community ro 0 private` | `% Invalid input` | El `no` no acepta `ro` / `rw` |
| `no snmp-server community private` | `% Invalid input` | Falta el tipo de cadena (`0` o `7`) |
| `no snmp-server community ?` | Mostró `0` (sin cifrar) / `7` (oculta) | Reveló la sintaxis correcta |
| `no snmp-server community 0 private` | ✅ Aceptado | Sintaxis correcta de este firmware |
| `no snmp-server host 192.168.10.10 v2c public` | Sin error, **pero no borró nada** | El texto no coincidía con lo guardado (`publc`) |

### 3.4 Configuración final verificada
```
hostname SW1
!
snmp-server enable
snmp-server securityip disable
snmp-server host 192.168.10.10 v2c public
snmp-server community ro 0 public
snmp-server community rw 0 private
snmp-server enable traps
!
interface Vlan1
 ip address 192.168.10.2 255.255.255.0
!
ip default-gateway 192.168.10.1
```
Guardado con `write` → *"Write configuration successfully!"* ✅

### 3.5 Lecciones del CLI Maipu (SM3000)

| Regla | Ejemplo |
| :--- | :--- |
| Para **crear** una comunidad se indica permiso y tipo | `snmp-server community rw 0 private` |
| Para **borrarla** solo se indica tipo y nombre (sin `ro` / `rw`) | `no snmp-server community 0 private` |
| Para borrar algo hay que escribirlo **exactamente** como quedó guardado, incluidos los errores | `no snmp-server host 192.168.10.10 v2c publc` |
| Un `no` que no coincide **no avisa**: siempre verificar | `show running-config` |
| Ante `% Invalid input`, usar la ayuda contextual | `no snmp-server community ?` |

---

## 4. Laptop gestor — IP 192.168.10.10

### 4.1 Problema
`ipconfig` mostraba en Ethernet **`169.254.69.66`** ("configuración automática").

| Dato | Valor | Significado |
| :--- | :--- | :--- |
| DHCP habilitado | Sí | El adaptador estaba en "Obtener IP automáticamente" |
| IP | `169.254.69.66` | APIPA: no hay servidor DHCP en el laboratorio |
| Enlace | Up, 100 Mbps | El cable estaba bien |

### 4.2 Solución (PowerShell como administrador)
```powershell
netsh interface ipv4 set address name="Ethernet" static 192.168.10.10 255.255.255.0 192.168.10.1
```

### 4.3 Verificación
```
Adaptador de Ethernet Ethernet:
   Dirección IPv4. . . . . . . . . . . . . . : 192.168.10.10
   Máscara de subred . . . . . . . . . . . . : 255.255.255.0
   Puerta de enlace predeterminada . . . . . : 192.168.10.1
```
Firewall de Windows: desactivado en perfiles privado y público ✅

> Si aparece de nuevo `169.254.x.x` o "(Duplicada)", otra máquina ya usa esa IP.

---

## 5. Diagnóstico: R1 no respondía

### 5.1 Síntomas (primer chequeo)

| Prueba | Resultado |
| :--- | :--- |
| Ping a SW1 `192.168.10.2` | ✅ 3/3, 3 ms |
| SNMP a SW1 (`sysName.0`) | ✅ `SW1` |
| Ping a R1 `192.168.10.1` | ❌ Sin respuesta |
| ARP de `192.168.10.1` | ❌ `Incomplete` (nadie contesta en el cable) |
| Estado de puertos de SW1 (leído por SNMP, `ifOperStatus`) | Ethernet1/1 **up**, Ethernet1/2 **up**, 1/3 y 1/4 down |

### 5.2 Conclusión
Ethernet1/2 tenía enlace, pero **no con la `fastethernet0` de R1**: el cable estaba en otro puerto del router (módulo switch 0–7 o `fastethernet1`).

### 5.3 Comprobaciones recomendadas en R1
```
show ip interface brief     ← fastethernet0 debe estar up/up con 192.168.10.1
ping 192.168.10.2           ← prueba hacia el switch desde el router
```

### 5.4 Resultado tras corregir el cableado
```
ping 192.168.10.2  → 4/4, 2 ms, TTL=64   (SW1)
ping 192.168.10.1  → 4/4, 3 ms, TTL=255  (R1)
```

---

## 6. Verificación SNMP (GET `sysName.0` = `.1.3.6.1.2.1.1.5.0`)

| Agente | Comunidad | Respuesta |
| :--- | :--- | :--- |
| R1 `192.168.10.1` | `public` | **R1** ✅ |
| SW1 `192.168.10.2` | `public` | **SW1** ✅ |

---

## 7. Pendiente: pruebas en iReasoning

> En **Address** usar `192.168.10.1` / `192.168.10.2`, no la red `192.168.1.x` de la entrega. En **Advanced**: puerto 161, `public` / `private`, versión 2.

| # | Prueba | Procedimiento | Resultado esperado | Estado |
| :-- | :--- | :--- | :--- | :--- |
| 1 | Identidad | GET `.1.3.6.1.2.1.1.5.0` a cada equipo | `R1` / `SW1` | ✅ iReasoning: `sysName.0 = SW1` (192.168.10.2:161) y `R1` (192.168.10.1:161) |
| 2 | Interfaces | Walk `.1.3.6.1.2.1.2.2.1.2` en `192.168.10.2` | Ethernet1/2 = ifIndex **2** | ✅ `ifDescr.1–16` = Ethernet1/1–1/16, `ifDescr.3001` = Vlan1; el Walk llegó a EndOfMibView (4052 filas, incluye la rama Maipu `.1.3.6.1.4.1.5651`) |
| 3 | Flujo | `ping 192.168.10.1 -t -l 65000` + Poll nuevo `Trafico_SW1_Eth1-2` (`.1.3.6.1.2.1.2.2.1.16.2` y `.10.2`, cada 5 s) | Contadores suben ≈ 330 KB cada 5 s (≈ 530 kbps) | ✅ Ver detalle en 7.1 |
| 4 | Traps | Tools → Trap Receiver; desconectar el cable de Ethernet1/2 | `linkDown` / `linkUp` desde `192.168.10.2` | ✅ Ver detalle en 7.2 |
| 5 | SET | Con `ping 192.168.10.1 -t`: Set `.1.3.6.1.2.1.2.2.1.7.2` = Integer 2, luego = 1 | El ping se corta y luego vuelve | ✅ Ver detalle en 7.3 |

> ⚠️ **Nunca** hacer el SET sobre el índice **1** (Ethernet1/1): es el puerto de la laptop y se pierde la gestión del switch.

### 7.1 Resultado de la prueba de flujo

**Ping de carga (primera ronda):** 193 enviados / 193 recibidos, **0 % pérdida**, RTT mín 53 ms / máx 67 ms / media **53 ms**, TTL=255.

> La latencia (≈ 53 ms) es mayor que con el router de la entrega (≈ 18 ms): R1 es un equipo más antiguo (MyPower 6.1.7) y tarda más en reensamblar y responder paquetes ICMP de 65 000 bytes fragmentados.

> **Primer intento fallido:** el poll se creó pero no se inició (el botón decía *Poll*, no *Poll every 5 seconds*); dos capturas mostraban los mismos valores (10 946 633 / 11 101 024). Se corrigió pulsando **Poll**.

| Lectura (poll activo) | ifInOctets.2 | ifOutOctets.2 | Estado |
| :--- | :--- | :--- | :--- |
| A — con ping | 13 548 593 | 13 712 533 | tráfico activo |
| B — con ping | 15 148 977 | 15 313 781 | tráfico activo |
| C — tras `Ctrl + C` | 15 148 977 | 15 313 781 | **sin cambio** |

| Cálculo | Valor |
| :--- | :--- |
| Δ ifOutOctets.2 (A → B) | **1 601 248 bytes** |
| Δ ifInOctets.2 (A → B) | **1 600 384 bytes** (las respuestas del router, casi idénticas) |
| Equivalencia | ≈ 24 pings de 65 000 bytes (≈ 66 KB c/u en el cable con cabeceras de los fragmentos) |
| Tiempo estimado entre A y B | ≈ 25 s (≈ 1 ping por segundo) |
| Tasa estimada | 1 601 248 × 8 / 25 ≈ **512 kbps** |
| Sin ping (B → C) | Δ = 0 → **≈ 0 bps** |

> Las capturas no registran la hora exacta, por eso la tasa es una estimación. Para un cálculo exacto: anotar dos lecturas consecutivas del poll (separadas exactamente 5 s) y aplicar `bps = Δ × 8 / 5`.

### 7.2 Resultado de la prueba de traps

**Comprobación previa:** UDP 162 abierto por iReasoning (`javaw`); firewall desactivado en la red del laboratorio (perfil público).

**Error en el primer intento:** se desconectó el cable de **Ethernet1/1 (la laptop)** en lugar del de Ethernet1/2. El adaptador quedó en *Disconnected* y el ping a SW1 y R1 falló: el switch genera el linkDown, pero **no tiene camino** para entregarlo al gestor (SNMP usa UDP, sin reintentos). Por eso la prueba siempre se hace sobre **otro enlace**, nunca el de gestión.

**Traps recibidos:**

| Hora | Source | OID | Trap | Causa |
| :--- | :--- | :--- | :--- | :--- |
| 22:13:23 | SW1 `192.168.10.2` | `.1.3.6.1.6.3.1.1.5.4` | **linkUp** | Se reconectó el cable de la laptop (Ethernet1/1) |
| 22:13:54 | SW1 `192.168.10.2` | `.1.3.6.1.6.3.1.1.5.3` | **linkDown** | Se desconectó el cable hacia R1 (Ethernet1/2) |
| 22:14:17 | SW1 `192.168.10.2` | `.1.3.6.1.6.3.1.1.5.4` | **linkUp** | Se reconectó Ethernet1/2 |
| 22:14:17 | R1 `192.168.10.1` | `.1.3.6.1.6.3.1.1.5.4` | **linkUp** | `fastethernet0` de R1 volvió a subir |

**Análisis:**
- El enlace estuvo caído **23 segundos** (22:13:54 → 22:14:17) y el gestor lo supo sin consultar: los agentes **notificaron** (modelo *push*).
- El **linkDown de R1 no llegó**, como se esperaba: su único camino hacia la laptop era el cable desconectado.
- El linkUp de R1 llegó en el **mismo segundo** que el del switch (en la entrega llegó ≈ 3 s después).
- El linkUp de 22:13:23 es un "bonus": demuestra que los traps del puerto de la laptop también funcionan, porque se envían **después** de recuperar el enlace.

### 7.3 Resultado de la prueba SET

OID: `.1.3.6.1.2.1.2.2.1.7.2` (`ifAdminStatus.2`, Ethernet1/2 hacia R1), agente `192.168.10.2`, comunidad de escritura `private`.

| # | Operación | Valor leído en la tabla de iReasoning |
| :-- | :--- | :--- |
| 1 | GET (antes) | `up (1)` |
| 2 | SET = Integer **2** → GET | `down (2)` — puerto apagado desde el software |
| 3 | SET = Integer **1** → GET | `up (1)` — puerto restaurado |
| 4 | GET de confirmación | `up (1)` |

**Efecto en la red (`ping 192.168.10.1 -t`):**

| Fase | Salida del ping |
| :--- | :--- |
| Antes del SET | `Respuesta desde 192.168.10.1: bytes=32 tiempo=2–3ms TTL=255` |
| Tras SET = 2 | 18 × `Tiempo de espera agotado` y 1 × `Respuesta desde 192.168.10.10: Host de destino inaccesible` |
| Tras SET = 1 | `Respuesta desde 192.168.10.1: bytes=32 tiempo=2–4ms TTL=255` |

> *"Host de destino inaccesible"* lo responde la **propia laptop** (192.168.10.10): al expirar la entrada ARP de R1, intentó resolver su MAC y nadie contestó, porque el único camino (Ethernet1/2) estaba apagado.

**Verificación independiente desde la laptop (tras la prueba):** `ifAdminStatus.2 = 1`, `ifOperStatus.2 = 1` y ping a R1 OK.

> Esta prueba confirma que la corrección de la comunidad (`ro 0 private` → `rw 0 private`, sección 3.3) era necesaria: con la configuración original el SET habría sido rechazado.

---

## 7.4 MIBs de Maipu

**Carga en iReasoning ✅:** tras *File → Load MIBs* (MAIPU-SMI y MPIOS-MIB), el árbol muestra `iso.org.dod.internet → private → enterprises → maipu` con las ramas `mpProducts`, `mpTrapObject`, `mpMgmt`, `mpExperiment`, `mpSecurity` y `mpMgmt2`.

**Identificación de los equipos por SNMP:**

| Agente | sysDescr | sysObjectID |
| :--- | :--- | :--- |
| R1 `192.168.10.1` | MyPower OS **MP1800**, versión 6.1.7 (2008) | `.1.3.6.1.4.1.5651.1.101.23` |
| SW1 `192.168.10.2` | **SM3000-16T** (S3016B V2), software 6.1.94.131 | `.1.3.6.1.4.1.5651.1.102.70` |

**Archivos:** `Lab-SMNP/mibs/maipu/MAIPU-SMI.mib` y `MPIOS-MIB.mib` (cargar en ese orden con **File → Load MIBs**).

**GET con nombre resuelto por la MIB de Maipu ✅ (iReasoning):**

| Name/OID | Value | Type | IP:Port |
| :--- | :--- | :--- | :--- |
| `numBytesFree.0` | 66 523 040 | Gauge | 192.168.10.1:161 |

El gestor muestra el **nombre** del objeto propietario (no solo el OID numérico), lo que prueba que la MIB quedó cargada. El árbol expandido muestra `maipu → mpMgmt → mpios → iosSystem → iosObjects` con los grupos `sysMemory`, `sysTask`, `sysCpu`, `sysTemperature`, `sysNFI` y `sysIfStatistic`.

**Objetos Maipu verificados (MPIOS-MIB):**

| Agente | Objeto | OID | Valor |
| :--- | :--- | :--- | :--- |
| R1 | `numBytesFree.0` (memoria libre) | `.1.3.6.1.4.1.5651.3.20.1.1.1.1.0` | 66 523 040 bytes ✅ |
| R1 | `numBytesAlloc.0` (memoria asignada) | `.1.3.6.1.4.1.5651.3.20.1.1.1.3.0` | 335 974 bytes ✅ |
| SW1 | `numBytesFree.0` | mismo OID | Sin respuesta (el SM3000 usa otra rama propietaria) |

---

## 8. Bitácora (20-nov-2026)

| # | Acción | Resultado |
| :-- | :--- | :--- |
| 1 | `show running-config` en SW1 (parte inferior) | Vlan1 `192.168.10.2/24`, sin gateway |
| 2 | `show running-config` en R1 | SNMP completo, traps a `192.168.10.10`, guardado |
| 3 | Configuración SNMP en SW1 + `write` | Guardado, con 2 errores (`ro 0 private`, `publc`) |
| 4 | Corrección de la comunidad `private` | Tras 3 intentos: `no snmp-server community 0 private` + `rw 0 private` ✅ |
| 5 | Primer intento de corregir el host | `no ... v2c public` no borró nada (no coincidía) |
| 6 | Segundo intento de corregir el host | `no ... v2c publc` + `... v2c public` ✅ |
| 7 | IP de la laptop | Estaba en APIPA `169.254.69.66`; `netsh` → `192.168.10.10` ✅ |
| 8 | Ping y SNMP | SW1 responde; R1 no (ARP incompleto) |
| 9 | Corrección del cableado hacia R1 | Ping a R1 4/4 ✅ |
| 10 | SNMP GET `sysName.0` | `R1` y `SW1` ✅ |
| 11 | Walk `ifDescr` en SW1 | Ethernet1/2 = ifIndex 2 ✅ |
| 12 | Poll `Trafico_SW1_Eth1-2` (tras iniciarlo con *Poll*) | Δ ≈ 1,6 MB con ping (≈ 512 kbps); 0 sin ping ✅ |
| 13 | Trap Receiver (primero se desconectó por error el cable de la laptop) | linkDown/linkUp de SW1 y linkUp de R1 ✅ |
| 14 | SET `ifAdminStatus.2` = 2 y = 1 | down/up, ping cortado y restaurado ✅ |
| 15 | Carga de MAIPU-SMI + MPIOS-MIB y GET `numBytesFree.0` a R1 | 66 523 040 (Gauge), con nombre resuelto ✅ |
