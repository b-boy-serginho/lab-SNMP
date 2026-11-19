# Guía, Auditoría e Informe — Laboratorio Gestor SNMP (Red 192.168.1.0/24)

**Materia:** Redes 2  
**Docente:** Gonzales  
**Práctica:** Laboratorio Gestor SNMP (grupo de 2 personas)  
**Entrega y defensa:** viernes 20 de noviembre de 2026 (horario de clases)  
**Gestor SNMP:** iReasoning MIB Browser (Portátil ASUS)  
**Última actualización:** 18 de noviembre de 2026

---

## Estado de avance (resumen)

| # | Etapa | Estado | Evidencia |
| :-- | :--- | :--- | :--- |
| 0 | Topología y direccionamiento definidos | ✅ Completado | Sección 0 |
| 1 | Agentes SNMP en router y switch | ✅ Configurados (auditados por `show running-config`) | Sección 1 |
| A | Laptop gestor en `192.168.1.10`, firewall desactivado | ✅ Completado y verificado | Sección 2 |
| B | Ajustes opcionales del switch (hostname, limpieza, descriptions) | ⏳ Pendiente | Sección 3 |
| C | Ajustes opcionales del router (hostname, description) | ⏳ Pendiente | Sección 4 |
| D1 | iReasoning: agente switch + `GET sysName.0` | ✅ Verificado (`switch`) | Sección 5.3 |
| D2 | iReasoning: `Walk ifDescr` → índice Ethernet1/2 | ✅ Verificado (`IDX = 2`) | Sección 5.4 |
| D3 | iReasoning: agente router + `GET sysName.0` | ✅ Verificado (`router`) | Sección 5.3 |
| D4 | Carga de MIBs de Maipu | ⏳ Archivos descargados en `mibs/maipu/`; falta cargarlos en iReasoning | Sección 5.1 |
| P2 | Prueba de flujo (Poll cada 5 s + cálculo de bps) | ✅ Verificado (≈534 kbps con ping / ≈0 sin ping) | Sección 6 |
| P3 | Prueba de errores (traps `linkDown`/`linkUp`) | ✅ Verificado (switch linkDown + linkUp, router linkUp) | Sección 6 |
| P4 | Prueba SET (`ifAdminStatus.2 = 2` y restauración con `1`) | ✅ Verificado (ping cortado y recuperado, trap linkDown) | Sección 6 |
| I | Informe final | ⏳ Pendiente | Sección 9 |

---

## 0. Topología y direccionamiento final

Equivalencia entre la gráfica del docente (Packet Tracer) y el laboratorio físico Maipu:

```
[Laptop ASUS - Gestor SNMP]           [Switch Maipu S3000]              [Router Maipu MP2804]
   192.168.1.10  ───────────── Ethernet1/1        Ethernet1/2 ───────────── fastethernet0
                  (ifIndex 1)   Vlan1 = 192.168.1.2   (ifIndex 2)            192.168.1.1
```

| Dispositivo | Gráfica docente | Interfaz física | Dirección IP | Máscara | Gateway | MAC (verificada) | Rol |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Router Maipu MP2804 | Router 1841 Fa0/0 | `fastethernet0` | `192.168.1.1` | `255.255.255.0` | — | `00-01-7A-AD-B1-D4` | Agente SNMP |
| Switch Maipu S3000 | Switch 2950 | `Vlan1` (SVI, ifIndex 3001) | `192.168.1.2` | `255.255.255.0` | `192.168.1.1` | `00-01-7A-F1-45-D3` | Agente SNMP |
| Laptop ASUS | Laptop-PT "SNMP SERVER" Fa0 | `Ethernet` (Realtek PCIe GbE) → Switch `Ethernet1/1` | `192.168.1.10` | `255.255.255.0` | `192.168.1.1` | — | Gestor SNMP + Trap Receiver |
| Laptop compañero (opcional) | — | Switch, puerto libre | `192.168.1.20` | `255.255.255.0` | `192.168.1.1` | — | Gestor secundario |

| Enlace | Origen | Destino |
| :--- | :--- | :--- |
| Gestión | Laptop `Ethernet` | Switch `Ethernet1/1` (= Fa0/1 de la gráfica) |
| Troncal | Switch `Ethernet1/2` (= Fa0/2 de la gráfica) | Router `fastethernet0` (= Fa0/0 de la gráfica) |

> **Nota:** Las notas anteriores (`feedback_*.md`) usaban `192.168.10.0/24`. La configuración real actual de los equipos está en `192.168.1.0/24` y los traps ya apuntan a `192.168.1.10`, por eso **solo se cambió la IP de la laptop**.
>
> **Dos laptops en la misma red no generan conflicto** mientras tengan IPs distintas (`.10` y `.20`). Ambas pueden hacer GET/SET; solo `192.168.1.10` recibe traps (ver sección 3.1 para agregar `.20`).

---

## 1. Auditoría de configuración actual de los equipos (según `show running-config`)

### Router (`hostname router`)
```
interface fastethernet0
 ip address 192.168.1.1 255.255.255.0
!
interface fastethernet1
 (sin configuración)
!
snmp-server start
snmp-server view default 1.3.6.1 include
snmp-server view TodoElArbol 1.3 include
snmp-server view readonlyview 1.3.6.1 include
snmp-server view readwriteview 1.3.6.1 include
snmp-server community public view default ro
snmp-server community private view default rw
snmp-server community RedesMaipuRo view TodoElArbol ro
snmp-server host 192.168.1.10 traps community public version 2
snmp-server enable traps snmp linkdown
snmp-server enable traps snmp linkup
... (más traps habilitados: authentication, coldstart, warmstart, ospf, bgp, frame-relay, etc.)
```

### Switch (`hostname switch`)
```
snmp-server enable
snmp-server securityip disable
snmp-server host 192.168.10.20 v2c public      <- sobrante (red antigua)
snmp-server host 192.168.1.10 v2c public
snmp-server community rw 0 RedesMaipuRw
snmp-server community ro 0 RedesMaipuRo
snmp-server community ro 0 public
snmp-server community rw 0 private
snmp-server enable traps
!
interface Vlan1
 ip address 192.168.1.2 255.255.255.0
!
ip default-gateway 192.168.1.1
```

### Resultado de la auditoría

| Punto revisado | Router | Switch | Observación |
| :--- | :--- | :--- | :--- |
| Servicio SNMP activo | ✅ `snmp-server start` | ✅ `snmp-server enable` | — |
| Comunidad lectura `public` | ✅ | ✅ | — |
| Comunidad escritura `private` | ✅ | ✅ | Necesaria para SET |
| Traps `linkDown`/`linkUp` habilitados | ✅ | ✅ | — |
| Destino de traps = gestor `192.168.1.10` | ✅ | ✅ | — |
| Destino de traps obsoleto | — | ⚠️ `192.168.10.20` | Eliminar (sección 3) |
| Restricción de IP de gestor | — | ✅ `securityip disable` | Permite cualquier gestor |
| Hostname descriptivo | ⚠️ `router` | ⚠️ `switch` | Opcional: `ROUTER_GESTIONADO` / `SWITCH_GESTIONADO` |
| Configuración guardada (`write`) | — | — | Hacer `write` tras cualquier cambio |

---

## 2. Paso A — Configurar la laptop (Gestor) ✅

### 2.1 Aislar la red
- [x] **Firewall de Windows** desactivado en *Redes privadas* y *Redes públicas* (Ethernet aparece como red pública "Red 2").
- [x] **Tailscale** sin conexión (`169.254.x.x`).
- [ ] **Wi-Fi** aún conectado (hotspot de celular). No interfiere con `192.168.1.x`, pero se recomienda apagarlo para la defensa:
  ```powershell
  netsh interface set interface name="Wi-Fi" admin=disable
  ```

Alternativa al firewall desactivado (abrir solo SNMP):
```powershell
New-NetFirewallRule -DisplayName "SNMP Lab" -Direction Inbound -Protocol UDP -LocalPort 161,162 -Action Allow
```

### 2.2 IP estática
Configurada en *Propiedades de IPv4* del adaptador `Ethernet` (o por PowerShell como administrador):

```powershell
netsh interface ipv4 set address name="Ethernet" static 192.168.1.10 255.255.255.0 192.168.1.1
```

| Parámetro | Valor |
| :--- | :--- |
| Dirección IP | `192.168.1.10` |
| Máscara | `255.255.255.0` |
| Puerta de enlace | `192.168.1.1` |
| DNS | (vacío, no se usa) |

> Inicialmente se colocó `192.168.1.20`; se corrigió a `.10` porque los traps del router y switch ya están configurados hacia `192.168.1.10`.

### 2.3 Verificación (resultado real)

| Prueba | Resultado |
| :--- | :--- |
| `ipconfig` → Ethernet | `192.168.1.10 / 255.255.255.0`, gateway `192.168.1.1` ✅ |
| `ping 192.168.1.2` (switch) | 3/3 respuestas, ~2.7 ms ✅ |
| `ping 192.168.1.1` (router) | 3/3 respuestas, ~2.3 ms ✅ |

> 📸 **Captura para el informe:** `ipconfig` + ambos pings exitosos.

### 2.4 Al terminar el laboratorio (restaurar)
```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
netsh interface set interface name="Wi-Fi" admin=enable
```

---

## 3. Paso B — Ajustes en el Switch Maipu S3000 (consola PuTTY) ⏳

> Opcional pero recomendado. Si algún comando no es aceptado, usar `?` (ej. `interface ?`) para ver la sintaxis; puede requerir espacio: `interface ethernet 1/1`.

```
config
hostname SWITCH_GESTIONADO
no snmp-server host 192.168.10.20 v2c public
interface ethernet1/1
description HACIA_GESTOR
exit
interface ethernet1/2
description HACIA_ROUTER
exit
exit
write
```

| Comando | Para qué sirve |
| :--- | :--- |
| `hostname SWITCH_GESTIONADO` | Nombre que verá el gestor en `sysName.0` |
| `no snmp-server host 192.168.10.20 ...` | Elimina destino de traps de la red antigua |
| `description ...` | Identifica el puerto en el gestor (`ifAlias`) |
| `write` | Guarda en Flash (*Write configuration successfully!*) |

Verificación:
```
show running-config
```
- [ ] `hostname SWITCH_GESTIONADO`
- [ ] Ya **no** aparece `snmp-server host 192.168.10.20`
- [ ] Sigue `snmp-server host 192.168.1.10 v2c public`
- [ ] Siguen `community ro 0 public` y `community rw 0 private`
- [ ] `interface Vlan1` con `192.168.1.2`

### 3.1 (Opcional) Traps también para la laptop del compañero `192.168.1.20`
```
config
snmp-server host 192.168.1.20 v2c public
exit
write
```

> 📸 **Captura para el informe:** bloque `snmp-server ...` del `show running-config`.

---

## 4. Paso C — Ajustes en el Router Maipu MP2804 (consola PuTTY) ⏳

```
configure terminal
hostname ROUTER_GESTIONADO
interface fastethernet0
description HACIA_SWITCH
exit
interface fastethernet1
shutdown
exit
exit
write
```

Verificación:
```
show running-config
show ip interface brief
ping 192.168.1.2
ping 192.168.1.10
```

### 4.1 (Opcional) Traps también para `192.168.1.20`
```
configure terminal
snmp-server host 192.168.1.20 traps community public version 2
exit
write
```

> 📸 **Captura para el informe:** bloque `snmp-server ...` del `show running-config`.

### 4.2 Comparativa de sintaxis (para el informe)

La guía del docente usa sintaxis genérica; en los equipos Maipu reales los comandos equivalentes son:

| Función | Guía del docente | Router MP2804 | Switch S3000 |
| :--- | :--- | :--- | :--- |
| Activar SNMP | `snmp-server agent` | `snmp-server start` | `snmp-server enable` |
| Comunidad lectura | `snmp-server community public ro` | `snmp-server community public view default ro` | `snmp-server community ro 0 public` |
| Comunidad escritura | `snmp-server community private rw` | `snmp-server community private view default rw` | `snmp-server community rw 0 private` |
| Vista MIB | — | `snmp-server view default 1.3.6.1 include` (obligatoria) | No requiere |
| Traps link | `snmp-server enable traps snmp linkdown linkup` | `snmp-server enable traps snmp linkdown` / `linkup` | `snmp-server enable traps` |
| Destino traps | — | `snmp-server host 192.168.1.10 traps community public version 2` | `snmp-server host 192.168.1.10 v2c public` |
| Guardar | — | `write` | `write` |

---

## 5. Paso D — Configurar iReasoning MIB Browser

> **Orden recomendado:** primero el **switch** (`192.168.1.2`), porque ahí se hacen el Walk, la prueba de flujo y el SET; luego el **router** (`192.168.1.1`) solo con GET.

### 5.1 Cargar MIBs (requisito del docente)

**Archivos obtenidos** (Maipu no publica sus MIBs; se descargaron del repositorio open source de LibreNMS, `mibs/maipu`):

| Archivo local | Origen | Tamaño | Contenido |
| :--- | :--- | :--- | :--- |
| `Lab-SMNP/mibs/maipu/MAIPU-SMI.mib` | `raw.githubusercontent.com/librenms/librenms/master/mibs/maipu/MAIPU-SMI` | 3.5 KB | Raíz `maipu ::= { enterprises 5651 }` y sub-ramas `mpProducts`, `mpMgmt`, `mpMgmt2`… |
| `Lab-SMNP/mibs/maipu/MPIOS-MIB.mib` | `raw.githubusercontent.com/librenms/librenms/master/mibs/maipu/MPIOS-MIB` | 81 KB | Sistema MyPower: memoria (`sysMemory`), tareas, CPU (`sysCpu`), temperatura. Importa `mpMgmt` de MAIPU-SMI |

**Pasos en iReasoning:**
1. **File → Load MIBs**.
2. Ir a `C:\Universidad\REDES2\Lab-SMNP\mibs\maipu\` y cargar **primero `MAIPU-SMI.mib`**, luego **`MPIOS-MIB.mib`** (depende del primero).
3. En el árbol aparece `iso.org.dod.internet.private.enterprises.maipu` (5651) → `mpMgmt` → `mpios`.
4. `RFC1213-MIB` / `IF-MIB` (MIB-II) ya vienen cargadas (`iso.org.dod.internet.mgmt.mib-2`).

**Prueba con la MIB de Maipu (verificada desde la laptop):**

| Agente | Objeto | OID | Resultado |
| :--- | :--- | :--- | :--- |
| Router `192.168.1.1` | `numBytesFree.0` (memoria libre) | `.1.3.6.1.4.1.5651.3.20.1.1.1.1.0` | **64,067,960 bytes** ✅ |
| Router `192.168.1.1` | `numBytesAlloc.0` (memoria asignada) | `.1.3.6.1.4.1.5651.3.20.1.1.1.3.0` | **278,556 bytes** ✅ |
| Router `192.168.1.1` | CPU (`cpuUtilCurrentUtil`) | `.1.3.6.1.4.1.5651.3.20.1.1.3.5.1.10` (Walk) | probar en iReasoning |
| Switch `192.168.1.2` | `numBytesFree.0` | mismo OID | Sin respuesta: el S3000 implementa otra rama propietaria |

> El router MP2804 implementa la MPIOS-MIB (su sysObjectID `5651.1.101.18` cuelga de `mpProducts`). El switch SM3000 (`5651.1.102.70`) no responde a esta rama; para él se usan las MIB estándar (IF-MIB) y su sysObjectID igualmente se resuelve bajo `maipu.mpProducts` al tener cargada MAIPU-SMI.

> 📸 **Captura:** árbol MIB mostrando la rama de Maipu.

### 5.2 Configurar el agente (botón **Advanced...**)
1. En **Address:** escribir la IP del agente (¡**no** `192.168.10.x`, es la red antigua!).
2. Clic en **Advanced...** y completar:

| Campo | Switch | Router |
| :--- | :--- | :--- |
| Address | `192.168.1.2` | `192.168.1.1` |
| Port | `161` | `161` |
| Read Community | `public` | `public` |
| Write Community | `private` | `private` |
| SNMP Version | `2` | `2` |
| Timeout / Retries | por defecto | por defecto |

3. Clic en **OK**.

### 5.3 Validación de identidad (`GET sysName.0`)

| Campo | Valor |
| :--- | :--- |
| OID | `.1.3.6.1.2.1.1.5.0` |
| Operations | **Get** (no *Get Next*) |
| Acción | **Go** |

**Resultados obtenidos:**

| Agente | Name/OID | Value | Type | IP:Port | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Switch | `sysName.0` | `switch` | OctetString | `192.168.1.2:161` | ✅ Verificado |
| Router | `sysName.0` | `router` | OctetString | `192.168.1.1:161` | ✅ Verificado |

**Identificación de los equipos (`sysDescr.0` = `.1.3.6.1.2.1.1.1.0`, `sysObjectID.0` = `.1.3.6.1.2.1.1.2.0`):**

| Agente | sysDescr | sysObjectID |
| :--- | :--- | :--- |
| Router `192.168.1.1` | MyPower OS — **MP2804** version 6.1.37, Maipu (Sichuan) Communication Technology | `.1.3.6.1.4.1.`**5651**`.1.101.18` |
| Switch `192.168.1.2` | **SM3000-16T** (S3016B V2), Software 6.1.94.131, Maipu Communication Technology | `.1.3.6.1.4.1.`**5651**`.1.102.70` |

> `5651` es el **número de empresa (Private Enterprise Number) de Maipu** asignado por IANA: todas las MIBs propietarias de Maipu cuelgan de `iso.org.dod.internet.private.enterprises.5651` (`.1.3.6.1.4.1.5651`).

> Si se aplica el paso B/C, el valor cambiará a `SWITCH_GESTIONADO` / `ROUTER_GESTIONADO`.

| Síntoma | Qué revisar |
| :--- | :--- |
| `Request timed out` | Address correcta, Read Community `public`, versión `2`, ping al equipo |
| `noSuchObject` | OID mal escrito (debe terminar en `.0`) |

> 📸 **Captura 1:** resultado `sysName.0` de cada agente.

### 5.4 Identificación de interfaces del switch (`Walk ifDescr`) ✅

| Campo | Valor |
| :--- | :--- |
| Address | `192.168.1.2` |
| OID | `.1.3.6.1.2.1.2.2.1.2` (`ifDescr`) |
| Operations | **Walk** |

**Resultado obtenido:**

| ifIndex | ifDescr | ifType | Uso en el laboratorio |
| :--- | :--- | :--- | :--- |
| **1** | `Ethernet1/1` | ethernetCsmacd (6) | Puerto del gestor — **NO apagar nunca** |
| **2** | **`Ethernet1/2`** | ethernetCsmacd (6) | **Enlace al router → prueba de flujo, traps y SET** |
| 3 – 16 | `Ethernet1/3` … `Ethernet1/16` | ethernetCsmacd (6) | Libres |
| 3001 | `Vlan1` | l3ipvlan (136) | Interfaz de gestión `192.168.1.2` |

> **`<IDX>` = 2** para todas las pruebas sobre el switch.
>
> Nota: el Walk continúa recorriendo todo el árbol (≈4094 filas). Detenerlo con el **botón rojo ✖** a la derecha de la tabla; limpiar la tabla con el ícono de tabla con ✖.

> 📸 **Captura 2:** tabla `ifDescr` con Ethernet1/1, Ethernet1/2 y Vlan1.

### 5.5 ¿De dónde salen los OIDs? (explicación para la defensa)

Los OIDs no se inventan: son rutas en el **árbol MIB** estandarizado. Cada número es un nodo del árbol:

```
.1        iso
 .3       org
  .6      dod
   .1     internet
    .2    mgmt
     .1   mib-2                        (RFC 1213, MIB-II)
      .1  system
       .1 sysDescr      .0 → instancia escalar
       .2 sysObjectID   .0
       .5 sysName       .0   ⇒ .1.3.6.1.2.1.1.5.0
      .2  interfaces
       .2 ifTable
        .1 ifEntry                     (RFC 2863, IF-MIB)
         .2  ifDescr       .<ifIndex>
         .7  ifAdminStatus .<ifIndex>  ⇒ .1.3.6.1.2.1.2.2.1.7.2  (puerto 2)
         .8  ifOperStatus  .<ifIndex>
         .10 ifInOctets    .<ifIndex>  ⇒ .1.3.6.1.2.1.2.2.1.10.2
         .14 ifInErrors    .<ifIndex>
         .16 ifOutOctets   .<ifIndex>  ⇒ .1.3.6.1.2.1.2.2.1.16.2
    .4    private
     .1   enterprises
      .5651 Maipu                      (MIBs propietarias de Maipu)
    .6    snmpV2
     .3.1.1.5 snmpTraps                (RFC 3418)
      .3  linkDown  ⇒ .1.3.6.1.6.3.1.1.5.3
      .4  linkUp    ⇒ .1.3.6.1.6.3.1.1.5.4
```

- **Objetos escalares** (un solo valor por equipo, ej. `sysName`) terminan en **`.0`**.
- **Objetos de tabla** (uno por interfaz, ej. `ifAdminStatus`) terminan en el **ifIndex** de la interfaz, obtenido con el Walk de `ifDescr` (sección 5.4).
- **Cómo verlos en iReasoning:** en el panel izquierdo expandir `iso.org.dod.internet.mgmt.mib-2` → `system` → `sysName` (o `interfaces` → `ifTable` → `ifEntry` → ...). Al hacer clic en un nodo, el panel inferior muestra **OID, MIB, Syntax y Access** (por ejemplo, `ifAdminStatus` aparece con *Access: read-write*, por eso admite SET; `ifInOctets` es *read-only*).
- **Fuentes:** RFC 1213 (MIB-II), RFC 2863 (IF-MIB), RFC 3418 (SNMPv2-MIB, traps genéricos).

### 5.6 Referencia rápida de OIDs usados (IF-MIB / MIB-II)

| Objeto | OID | Uso |
| :--- | :--- | :--- |
| `sysName.0` | `.1.3.6.1.2.1.1.5.0` | Identidad del agente |
| `ifDescr` | `.1.3.6.1.2.1.2.2.1.2` | Nombre de interfaces |
| `ifAdminStatus.2` | `.1.3.6.1.2.1.2.2.1.7.2` | **SET**: 1 = up, 2 = down |
| `ifOperStatus.2` | `.1.3.6.1.2.1.2.2.1.8.2` | Estado real: 1 = up, 2 = down |
| `ifInOctets.2` | `.1.3.6.1.2.1.2.2.1.10.2` | Bytes que entran a Ethernet1/2 |
| `ifInErrors.2` | `.1.3.6.1.2.1.2.2.1.14.2` | Errores de entrada |
| `ifOutOctets.2` | `.1.3.6.1.2.1.2.2.1.16.2` | Bytes que salen por Ethernet1/2 |

---

## 6. Paso E — Pruebas de verificación del docente

### Paso 1 del docente — Agentes configurados
Mostrar los `show running-config` del router y del switch (secciones 1, 3 y 4) y la comparativa de sintaxis (4.2).

### Paso 2 del docente — Flujo de tráfico (Poll) ✅

> En la edición **Free/Personal** de iReasoning, el clic derecho no ofrece *Graph* y las herramientas *Port View* / *Switch Port Mapper* están deshabilitadas. Se usó **Polls → Create Poll**, que consulta los contadores automáticamente cada 5 s.

1. En **CMD** de la laptop, dejar corriendo:
   ```cmd
   ping 192.168.1.1 -t -l 65000
   ```
   El tráfico entra al switch por Ethernet1/1 y sale por **Ethernet1/2** hacia el router.
2. En iReasoning: **Polls → Create Poll**:

   | Campo | Valor |
   | :--- | :--- |
   | Poll Name | `Trafico_Ethernet1-2` |
   | Interval | `5` seconds |
   | SNMP Agents | **Add Agent** → `192.168.1.2`, puerto 161, `public` / `private`, versión 2 |
   | Variable 1 | `ifOutOctets.2` — `.1.3.6.1.2.1.2.2.1.16.2` — GET |
   | Variable 2 | `ifInOctets.2` — `.1.3.6.1.2.1.2.2.1.10.2` — GET |

3. **Ok** → abrir **Polls → Trafico_Ethernet1-2** → pulsar **Poll** (queda *Poll every 5 seconds*). Los valores se actualizan solos.
4. Observar el crecimiento con ping; detener el ping (`Ctrl + C`) y observar que el contador prácticamente se detiene.
5. Cálculo: **bps = (V2 − V1) × 8 / 5**.

**Resultados obtenidos:**

| Momento | ifInOctets.2 | ifOutOctets.2 | Δ out (5 s) | bps |
| :--- | :--- | :--- | :--- | :--- |
| Con ping (lectura de control t=0) | — | 91,637,316 | — | — |
| Con ping (t=+5 s) | — | 91,970,937 | +333,621 B | **≈ 533.8 kbps** |
| Con ping (poll, captura A) | 97,685,100 | 98,106,716 | — | — |
| Con ping (poll, captura B, ~95 s después) | 104,086,636 | 104,519,026 | +6,412,310 B en ~95 s | ≈ 540 kbps |
| Sin ping (lectura 1) | 108,487,772 | 108,930,311 | — | — |
| Sin ping (lectura 2, +5 s) | 108,487,772 | 108,930,468 | +157 B | ≈ 0.25 kbps |
| Sin ping (lectura 3, +5 s) | 108,487,772 | 108,930,939 | +471 B | ≈ 0.75 kbps |

**Ping de carga:** 1622 paquetes de 65000 bytes enviados, 1622 recibidos, **0 % pérdida**, RTT mín 18 ms / máx 31 ms / media 18 ms, TTL=255.

**Interpretación:** con la ráfaga ICMP, el gestor mide ≈ **534 kbps** saliendo por Ethernet1/2 (≈ 65 000 B/s, un ping de 65 000 bytes por segundo) y un valor casi idéntico entrando (respuestas del router). Al detener el ping, `ifInOctets.2` queda congelado y `ifOutOctets.2` sólo sube unos cientos de bytes (tráfico de control: ARP, broadcast), es decir, ≈ **0 bps**. El gestor refleja en tiempo real el aumento y la caída del tráfico.

> 📸 **Captura 3:** ventana del Poll + CMD con el ping (con ráfaga y tras `Ctrl + C`).

### Paso 3 del docente — Detección de errores (Traps) ⏳
1. iReasoning → **Tools → Trap Receiver** (`Ctrl + I`). Verificar que escucha en UDP `162`.
2. Desconectar el cable de **Ethernet1/2** del switch (≈3 s) y volver a conectarlo.
3. Deben llegar:
   - Desde `192.168.1.2` (switch): `linkDown` y `linkUp` de ifIndex 2.
   - Desde `192.168.1.1` (router): `linkDown` y `linkUp` de `fastethernet0`.
4. Complementario: `GET ifOperStatus.2` → `2` (down) con cable desconectado / `1` (up) conectado; `GET ifInErrors.2`.

> Si no llegan traps: firewall (UDP 162), que la laptop sea `192.168.1.10`, que no haya otro programa usando el puerto 162.

**Resultados obtenidos (18-nov-2026):**

| Hora | Source | OID recibido | Trap | Significado |
| :--- | :--- | :--- | :--- | :--- |
| 21:47:00 | `192.168.1.2` (switch) | `.1.3.6.1.6.3.1.1.5.3` | **linkDown** | Se desconectó el cable de Ethernet1/2 |
| 21:47:36 | `192.168.1.2` (switch) | `.1.3.6.1.6.3.1.1.5.4` | **linkUp** | Se reconectó el cable de Ethernet1/2 |
| 21:47:39 | `192.168.1.1` (router) | `.1.3.6.1.6.3.1.1.5.4` | **linkUp** | fastethernet0 del router volvió a subir |

**Análisis (importante para la defensa):**
- Los OIDs son los traps genéricos estándar de SNMPv2 (`SNMPv2-MIB::snmpTraps`): `.5.3` = `linkDown`, `.5.4` = `linkUp`. iReasoning los muestra como OID porque la descripción depende de las MIBs cargadas.
- El **linkDown del router no llegó**, y es lo esperado: el router envía sus traps a `192.168.1.10` **a través del mismo cable desconectado** (router FE0 → switch Ethernet1/2 → laptop). Mientras el enlace está caído, ese trap no tiene camino al gestor (SNMP usa UDP, no hay reintento). El switch sí pudo avisar porque la laptop está conectada directamente a su puerto Ethernet1/1.
- El linkUp del router llega ~3 s después del linkUp del switch: tiempo de negociación del enlace FE0 y ARP antes de poder alcanzar al gestor.
- El gestor detectó la falla ~36 s antes de la recuperación, sin consultar activamente: los agentes **notificaron** el evento (modelo *push* de los traps frente al *pull* del GET/Poll).

> 📸 **Captura 4:** Trap Receiver con `linkDown` / `linkUp` (ya obtenida).

### Paso 4 del docente — Apagado remoto (SET) ⭐ ⏳
> ⚠️ **Apagar ifIndex 2 (Ethernet1/2, hacia el router). NUNCA ifIndex 1 (Ethernet1/1)**: es el puerto del gestor y se perdería la gestión del switch.

1. En **CMD**, dejar corriendo:
   ```cmd
   ping 192.168.1.1 -t
   ```
2. En iReasoning:
   - **Address:** `192.168.1.2`
   - **OID:** `.1.3.6.1.2.1.2.2.1.7.2` (`ifAdminStatus.2`)
   - **Operations:** **Set** → **Go**
   - Data Type: `Integer` — Value: **`2`** (down) → **OK**
3. Resultado esperado:
   - El ping al router pasa a *Tiempo de espera agotado*.
   - El LED del puerto 2 del switch se apaga.
   - (Bonus) Llega trap `linkDown` al Trap Receiver.
   - `ping 192.168.1.2` sigue respondiendo (el gestor no perdió el switch).
4. Restaurar: mismo OID → **Set** → `Integer` = **`1`** (up). El ping vuelve a responder.
5. (Opcional) En la consola del switch, `show interface ethernet1/2` muestra *administratively down* mientras está en `2`.

**Resultados obtenidos (18-nov-2026):**

| # | Operación en iReasoning | Resultado en el gestor | Efecto en la red |
| :-- | :--- | :--- | :--- |
| 1 | `GET ifAdminStatus.2` (antes) | `up (1)` | `ping 192.168.1.1 -t` responde (1–3 ms, TTL=255) |
| 2 | `SET ifAdminStatus.2` = Integer **2** | Aceptado (comunidad `private`) | Ping pasa a **"Tiempo de espera agotado"**; Trap Receiver recibe **linkDown** (`.1.3.6.1.6.3.1.1.5.3`) de `192.168.1.2` a las **21:58:43** |
| 3 | `GET ifAdminStatus.2` (después) | `down (2)` | Confirmado también `ifOperStatus.2 = 2` |
| 4 | `SET ifAdminStatus.2` = Integer **1** | Aceptado | Ping vuelve a responder (2–3 ms, TTL=255) |
| 5 | Verificación final | `ifAdminStatus.2 = 1`, `ifOperStatus.2 = 1` | Red restaurada |

**Análisis:**
- El puerto Ethernet1/2 se apagó **administrativamente desde el software** (equivalente a `shutdown` en la CLI) sin tocar la consola del switch, y el tráfico hacia el router se cortó de inmediato: se demuestra que iReasoning es un **gestor** (GET + SET) y no un simple monitor.
- El switch generó por sí mismo el trap `linkDown` al cambiar de estado la interfaz.
- El gestor nunca perdió el control del switch porque la laptop está en Ethernet1/1 (ifIndex 1), un puerto distinto al apagado; por eso fue posible restaurar con `SET = 1`.
- Precaución observada: tras el SET, el combo *Operations* debe cambiarse explícitamente a **Set** para restaurar; un *Get* sólo vuelve a leer `down (2)`.

> 📸 **Captura 5:** ventana SNMP SET (Integer 2) + CMD con el corte + tabla `up (1)` / `down (2)` + CMD con el ping recuperado.

---

## 7. Bitácora de auditoría (18-nov-2026)

| # | Acción | Resultado |
| :-- | :--- | :--- |
| 1 | Revisión de `show running-config` del router y switch | Agentes SNMP correctos en `192.168.1.0/24`; traps hacia `192.168.1.10`; destino obsoleto `192.168.10.20` en switch |
| 2 | Revisión de adaptadores de la laptop | Ethernet en `192.168.10.20` (red antigua), Wi-Fi y Tailscale activos |
| 3 | Cambio de IP de la laptop | Primero `192.168.1.20`, corregido a `192.168.1.10` |
| 4 | Ping a switch y router | 100% éxito (~2–3 ms) |
| 5 | Verificación ARP | Router `00-01-7A-AD-B1-D4` (`.1`), Switch `00-01-7A-F1-45-D3` (`.2`) — equipos distintos, sin IP duplicada |
| 6 | Firewall de Windows | Desactivado en perfiles privado y público |
| 7 | iReasoning: Address corregido de `192.168.10.2` a `192.168.1.2` | OK |
| 8 | `GET sysName.0` al switch | `switch` (OctetString) ✅ |
| 9 | `Walk ifDescr` al switch | Ethernet1/1–1/16 = ifIndex 1–16; Vlan1 = 3001; **Ethernet1/2 = 2** ✅ |
| 10 | Búsqueda de gráfica en iReasoning Free | *Graph* no disponible; *Port View* deshabilitado → se usa **Polls** |
| 11 | Poll `Trafico_Ethernet1-2` (5 s) con `ping -l 65000` | `ifOutOctets.2` crece ≈333 KB cada 5 s → **≈534 kbps** ✅ |
| 12 | Ping detenido (1622/1622, 0 % pérdida) | Contadores casi congelados (+157 / +471 B en 5 s) → ≈0 bps ✅ |
| 13 | Trap Receiver + desconexión del cable Ethernet1/2 | Switch: linkDown 21:47:00, linkUp 21:47:36; Router: linkUp 21:47:39 ✅ (linkDown del router no llega: su camino al gestor era el cable desconectado) |
| 14 | `SET ifAdminStatus.2 = 2` con ping activo | Ping cortado; trap linkDown 21:58:43; `GET` → `down (2)` ✅ |
| 15 | `SET ifAdminStatus.2 = 1` | Ping restaurado; verificación `ifAdminStatus.2 = 1`, `ifOperStatus.2 = 1` ✅ |
| 16 | `GET sysName.0` al router `192.168.1.1` | `router` (OctetString) ✅ |
| 17 | `GET sysDescr.0` / `sysObjectID.0` a ambos | Router MP2804 v6.1.37 (`5651.1.101.18`); Switch SM3000-16T v6.1.94.131 (`5651.1.102.70`) — enterprise Maipu = 5651 |
| 18 | MIBs de Maipu | Descargadas de LibreNMS: `MAIPU-SMI.mib` (3.5 KB) y `MPIOS-MIB.mib` (81 KB) en `Lab-SMNP/mibs/maipu/`; contenido revisado (texto ASN.1, sin código) |
| 19 | Prueba MPIOS-MIB | Router responde `numBytesFree.0 = 64,067,960`, `numBytesAlloc.0 = 278,556` ✅; switch no implementa esa rama |

---

## 8. Checklist final

- [x] Laptop en `192.168.1.10/24`, firewall desactivado, Tailscale desconectado.
- [ ] Wi-Fi apagado para la defensa.
- [x] Ping exitoso a `192.168.1.1` y `192.168.1.2`.
- [ ] Switch: hostname, limpieza de `192.168.10.20`, descriptions, `write`.
- [ ] Router: hostname, description, `write`.
- [ ] MIBs de Maipu cargadas en iReasoning.
- [x] `sysName.0` responde en el switch.
- [x] `sysName.0` responde en el router.
- [x] Walk de `ifDescr` hecho → Ethernet1/2 = ifIndex **2**.
- [x] Prueba de flujo (Poll + cálculo de bps).
- [x] Prueba de traps `linkDown` / `linkUp`.
- [x] Prueba SET `ifAdminStatus.2 = 2` y restauración con `1`.
- [ ] Todas las capturas guardadas para el informe.
- [ ] Al terminar: reactivar firewall y Wi-Fi.

---

## 9. Estructura del informe

1. **Carátula:** integrantes del grupo (2), materia, docente, fecha.
2. **Objetivo:** gestionar un router y un switch Maipu reales mediante un gestor SNMP capaz de leer (GET) y modificar (SET) la red.
3. **Selección del gestor:** iReasoning MIB Browser — permite **SET**, Walk, gráficas, Trap Receiver y carga de MIBs propietarias. Diferencia entre **gestor** (GET + SET) y **monitor** (solo GET, ej. PRTG).
4. **Topología y direccionamiento:** diagrama y tabla de la sección 0.
5. **Configuración de los agentes:** capturas `show running-config` (secciones 1, 3, 4) y **comparativa de sintaxis Maipu vs. guía** (4.2).
6. **Configuración del gestor:** IP de la laptop (2), parámetros de Advanced (5.2), carga de MIBs (5.1).
7. **Pruebas:**
   - Identidad (`sysName.0`) — 5.3
   - Identificación de interfaces (`ifDescr`) — 5.4
   - Flujo de tráfico (gráfica `ifOutOctets.2` / `ifInOctets.2`) — 6
   - Errores (traps `linkDown` / `linkUp`) — 6
   - SET (`ifAdminStatus.2`) — 6
   - Explicación de cada OID usado (5.5).
8. **Problemas encontrados y soluciones:**
   - Sintaxis SNMP distinta entre router y switch Maipu y respecto a la guía genérica.
   - Cambio de red `192.168.10.0/24` → `192.168.1.0/24`; IP del gestor alineada al destino de traps.
   - Firewall de Windows bloqueando UDP 161/162.
   - Walk que recorre todo el árbol (detener manualmente).
9. **Conclusiones.**
