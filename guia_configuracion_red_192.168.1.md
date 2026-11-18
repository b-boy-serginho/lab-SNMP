# Guía de Configuración — Laboratorio Gestor SNMP (Red 192.168.1.0/24)

**Materia:** Redes 2  
**Docente:** Gonzales  
**Práctica:** Laboratorio Gestor SNMP  
**Entrega y defensa:** viernes 20 de noviembre de 2026 (horario de clases)  
**Gestor SNMP:** iReasoning MIB Browser (Portátil ASUS)

---

## 0. Topología y direccionamiento final

Equivalencia entre la gráfica del docente (Packet Tracer) y el laboratorio físico Maipu:

```
[Laptop ASUS - Gestor SNMP]           [Switch Maipu S3000]              [Router Maipu MP2804]
   192.168.1.10  ───────────── Ethernet1/1        Ethernet1/2 ───────────── fastethernet0
                                  Vlan1 = 192.168.1.2                        192.168.1.1
```

| Dispositivo | Gráfica docente | Interfaz física | Dirección IP | Máscara | Gateway | Rol |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Router Maipu MP2804 | Router 1841 Fa0/0 | `fastethernet0` | `192.168.1.1` | `255.255.255.0` | — | Agente SNMP (`ROUTER_GESTIONADO`) |
| Switch Maipu S3000 | Switch 2950 | `Vlan1` (SVI) | `192.168.1.2` | `255.255.255.0` | `192.168.1.1` | Agente SNMP (`SWITCH_GESTIONADO`) |
| Laptop ASUS | Laptop-PT "SNMP SERVER" Fa0 | `Ethernet` → Switch `Ethernet1/1` | `192.168.1.10` | `255.255.255.0` | `192.168.1.1` | Gestor SNMP + Trap Receiver |

| Enlace | Origen | Destino |
| :--- | :--- | :--- |
| Gestión | Laptop `Ethernet` | Switch `Ethernet1/1` (= Fa0/1 de la gráfica) |
| Troncal | Switch `Ethernet1/2` (= Fa0/2 de la gráfica) | Router `fastethernet0` (= Fa0/0 de la gráfica) |

> **Nota:** Las notas anteriores (`feedback_*.md`) usaban `192.168.10.0/24`. La configuración real actual de los equipos está en `192.168.1.0/24` y los traps ya apuntan a `192.168.1.10`, por eso **solo se cambia la IP de la laptop**.

---

## 1. Estado actual de los equipos (según `show running-config`)

### Router (`hostname router`)
```
interface fastethernet0
 ip address 192.168.1.1 255.255.255.0
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
... (más traps habilitados: coldstart, warmstart, authentication, ospf, bgp, etc.)
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

**Diagnóstico:** ambos agentes SNMP están correctamente configurados (lectura `public`, escritura `private`, traps `linkDown`/`linkUp` hacia `192.168.1.10`). Solo falta ajustar la laptop y hacer una limpieza opcional.

---

## 2. Paso A — Configurar la laptop (Gestor)

### 2.1 Aislar la red
- [ ] Desactivar **Wi-Fi**.
- [ ] Cerrar **Tailscale** / cualquier VPN.
- [ ] Desactivar temporalmente el **Firewall de Windows** (perfiles privado y público) **o** crear reglas de entrada para **UDP 161 y UDP 162** e **ICMP**.

### 2.2 IP estática
Conectar el cable de la laptop al puerto **Ethernet1/1** del switch.

PowerShell **como administrador** (cambiar `"Ethernet"` por el nombre real del adaptador si es distinto — verlo con `Get-NetAdapter`):

```powershell
netsh interface ipv4 set address name="Ethernet" static 192.168.1.10 255.255.255.0 192.168.1.1
```

Alternativa gráfica: *Panel de control → Centro de redes → Cambiar configuración del adaptador → Ethernet → Propiedades → IPv4*.

### 2.3 Verificar conectividad
```powershell
ipconfig
ping 192.168.1.2
ping 192.168.1.1
```
Resultado esperado: 0% de pérdida hacia ambos (TTL=64 switch, TTL=255 router).

> 📸 **Captura para el informe:** `ipconfig` + ambos pings exitosos.

---

## 3. Paso B — Ajustes en el Switch Maipu S3000 (consola PuTTY)

> Opcional pero recomendado: deja la configuración limpia y los puertos identificados en el gestor (`ifAlias`). Si algún comando no es aceptado, usar `?` para ver la sintaxis correcta.

```
switch# config
switch(config)# hostname SWITCH_GESTIONADO
SWITCH_GESTIONADO(config)# no snmp-server host 192.168.10.20 v2c public
SWITCH_GESTIONADO(config)# interface ethernet1/1
SWITCH_GESTIONADO(Config-If-Ethernet1/1)# description HACIA_GESTOR
SWITCH_GESTIONADO(Config-If-Ethernet1/1)# exit
SWITCH_GESTIONADO(config)# interface ethernet1/2
SWITCH_GESTIONADO(Config-If-Ethernet1/2)# description HACIA_ROUTER
SWITCH_GESTIONADO(Config-If-Ethernet1/2)# exit
SWITCH_GESTIONADO(config)# exit
SWITCH_GESTIONADO# write
```

Verificación:
```
SWITCH_GESTIONADO# show running-config
SWITCH_GESTIONADO# show interface ethernet1/2
```

> 📸 **Captura para el informe:** bloque `snmp-server ...` del `show running-config`.

---

## 4. Paso C — Ajustes en el Router Maipu MP2804 (consola PuTTY)

```
router# configure terminal
router(config)# hostname ROUTER_GESTIONADO
ROUTER_GESTIONADO(config)# interface fastethernet0
ROUTER_GESTIONADO(config-if-fastethernet0)# description HACIA_SWITCH
ROUTER_GESTIONADO(config-if-fastethernet0)# exit
ROUTER_GESTIONADO(config)# interface fastethernet1
ROUTER_GESTIONADO(config-if-fastethernet1)# shutdown
ROUTER_GESTIONADO(config-if-fastethernet1)# exit
ROUTER_GESTIONADO(config)# exit
ROUTER_GESTIONADO# write
```

Verificación:
```
ROUTER_GESTIONADO# show running-config
ROUTER_GESTIONADO# show ip interface brief
ROUTER_GESTIONADO# ping 192.168.1.2
ROUTER_GESTIONADO# ping 192.168.1.10
```

> 📸 **Captura para el informe:** bloque `snmp-server ...` del `show running-config`.

### Comparativa de sintaxis (para el informe)

La guía del docente usa sintaxis genérica; en los equipos Maipu reales los comandos equivalentes son:

| Función | Guía del docente | Router MP2804 | Switch S3000 |
| :--- | :--- | :--- | :--- |
| Activar SNMP | `snmp-server agent` | `snmp-server start` | `snmp-server enable` |
| Comunidad lectura | `snmp-server community public ro` | `snmp-server community public view default ro` | `snmp-server community ro 0 public` |
| Comunidad escritura | `snmp-server community private rw` | `snmp-server community private view default rw` | `snmp-server community rw 0 private` |
| Traps link | `snmp-server enable traps snmp linkdown linkup` | `snmp-server enable traps snmp linkdown` / `linkup` | `snmp-server enable traps` |
| Destino traps | — | `snmp-server host 192.168.1.10 traps community public version 2` | `snmp-server host 192.168.1.10 v2c public` |
| Guardar | — | `write` | `write` |

---

## 5. Paso D — Configurar iReasoning MIB Browser

### 5.1 Cargar MIBs (requisito del docente)
1. **File → Load MIBs**.
2. Cargar los archivos `.mib` de **Maipu** (CD del equipo / web de Maipu / solicitados al docente).
3. Verificar que `IF-MIB` y `RFC1213-MIB` (MIB-II) estén cargadas (vienen por defecto).

> 📸 **Captura:** árbol MIB mostrando la rama de Maipu (`enterprises`).

### 5.2 Agentes
En el campo **Address** → botón **Advanced…**:

| Parámetro | Switch | Router |
| :--- | :--- | :--- |
| Address | `192.168.1.2` | `192.168.1.1` |
| Port | `161` | `161` |
| SNMP Version | `2` | `2` |
| Read Community | `public` | `public` |
| Write Community | `private` | `private` |

### 5.3 Validación de identidad (`GET`)
- OID: `.1.3.6.1.2.1.1.5.0` (`sysName.0`) → **Operations: Get**.
- Esperado: `SWITCH_GESTIONADO` / `ROUTER_GESTIONADO`.

### 5.4 Identificar el índice de las interfaces del switch (**obligatorio antes del SET**)
- Address `192.168.1.2`, OID `.1.3.6.1.2.1.2.2.1.2` (`ifDescr`) → **Operations: Walk**.
- Anotar qué índice corresponde a cada puerto:

| Puerto | ifIndex (completar) |
| :--- | :--- |
| `Ethernet1/1` (gestor) | `____` |
| `Ethernet1/2` (router) | `____` |

> En adelante, `<IDX>` = índice de **Ethernet1/2**.

---

## 6. Paso E — Pruebas de verificación del docente

### Prueba 1 / Paso 1 — Agentes configurados
Mostrar los `show running-config` del router y del switch (secciones 3 y 4).

### Prueba 2 / Paso 2 — Flujo de tráfico (GET)
1. En CMD de la laptop:
   ```cmd
   ping 192.168.1.1 -t -l 65000
   ```
2. En iReasoning: Address `192.168.1.2`, OID `.1.3.6.1.2.1.2.2.1.10.<IDX>` (`ifInOctets`) → clic derecho → **Graph** (o *Operations → Graph*), intervalo 5 s.
3. Repetir con `.1.3.6.1.2.1.2.2.1.16.<IDX>` (`ifOutOctets`).
4. Observar cómo sube la gráfica al iniciar el ping y cae al detenerlo (`Ctrl + C`).

| Momento | ifInOctets.<IDX> | Observación |
| :--- | :--- | :--- |
| Reposo | | |
| Durante ráfaga | | |
| Fin de ráfaga | | |

> 📸 **Captura:** gráfica con el pico de tráfico + CMD con el ping.

### Prueba 3 / Paso 3 — Detección de errores (Traps)
1. iReasoning → **Tools → Trap Receiver** (`Ctrl + I`). Verificar que escucha en UDP `162`.
2. Desconectar el cable de **Ethernet1/2** del switch (≈3 s) y volver a conectarlo.
3. Deben llegar:
   - Desde `192.168.1.2` (switch): `linkDown` y `linkUp`.
   - Desde `192.168.1.1` (router): `linkDown` y `linkUp` de `fastethernet0`.
4. Opcional: consultar `ifOperStatus.<IDX>` (`.1.3.6.1.2.1.2.2.1.8.<IDX>`) → `2` (down) desconectado / `1` (up) conectado, e `ifInErrors.<IDX>` (`.1.3.6.1.2.1.2.2.1.14.<IDX>`).

> Si no llegan traps: revisar firewall (UDP 162), que la laptop tenga `192.168.1.10` y que no haya otro programa usando el puerto 162.

> 📸 **Captura:** ventana Trap Receiver con `linkDown` / `linkUp`.

### Prueba 4 / Paso 4 — Apagado remoto (SET) ⭐
> ⚠️ **Apagar Ethernet1/2 (hacia el router), NUNCA Ethernet1/1** (es el puerto del gestor; se perdería la gestión del switch).

1. En CMD dejar corriendo:
   ```cmd
   ping 192.168.1.1 -t
   ```
2. En iReasoning: Address `192.168.1.2`, OID `.1.3.6.1.2.1.2.2.1.7.<IDX>` (`ifAdminStatus`) → **Operations: Set**.
   - Data Type: `Integer`
   - Value: **`2`** (down) → **OK**
3. Resultado esperado:
   - El ping al router empieza a dar *Tiempo de espera agotado*.
   - El LED del puerto 2 del switch se apaga.
   - (Bonus) Llega trap `linkDown` al Trap Receiver.
   - `ping 192.168.1.2` sigue respondiendo (el gestor no perdió el switch).
4. Restaurar: mismo OID → **Set** → `Integer` = **`1`** (up). El ping vuelve a responder.
5. (Opcional) Verificar en la consola del switch que el puerto quedó en `administratively down` mientras estuvo en `2`.

> 📸 **Captura:** ventana del SET + CMD mostrando el corte y la recuperación del ping.

---

## 7. Checklist final

- [ ] Laptop en `192.168.1.10/24`, Wi-Fi/VPN apagados, firewall ajustado.
- [ ] Ping exitoso a `192.168.1.1` y `192.168.1.2`.
- [ ] Switch: hostname, limpieza de `192.168.10.20`, descriptions, `write`.
- [ ] Router: hostname, description, `write`.
- [ ] MIBs de Maipu cargadas en iReasoning.
- [ ] `sysName.0` responde en ambos agentes.
- [ ] Walk de `ifDescr` hecho y `<IDX>` de Ethernet1/2 identificado.
- [ ] Prueba de flujo con gráfica.
- [ ] Prueba de traps `linkDown` / `linkUp`.
- [ ] Prueba SET `ifAdminStatus = 2` y restauración con `1`.
- [ ] Todas las capturas guardadas para el informe.

## 8. Estructura sugerida del informe

1. Carátula (integrantes del grupo de 2, materia, docente, fecha).
2. Objetivo.
3. Selección del gestor (por qué iReasoning: permite **SET** y carga de MIBs; diferencia gestor vs. monitor tipo PRTG).
4. Topología y tabla de direccionamiento (sección 0).
5. Configuración de los agentes + comparativa de sintaxis Maipu (secciones 3 y 4).
6. Configuración del gestor y carga de MIBs (sección 5).
7. Pruebas: flujo, errores, SET (sección 6) con capturas y explicación de los OIDs usados.
8. Problemas encontrados y soluciones (sintaxis distinta entre modelos, firewall, cambio de red).
9. Conclusiones.
