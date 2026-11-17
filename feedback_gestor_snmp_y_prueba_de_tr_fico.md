# Feedback Técnico — Configuración del Gestor SNMP (Portátil ASUS) y Prueba de Flujo de Tráfico

**Materia:** Redes 2  
**Docente:** Gonzales  
**Práctica:** Laboratorio Gestor SNMP  
**Estación de Gestión:** Portátil ASUS (`192.168.10.20`)  
**Software Utilizado:** iReasoning MIB Browser (Free Personal Edition)  

---

## 1. Diagnóstico y Aislamiento de Red en la Estación de Gestión

Para evitar conflictos de enrutamiento con redes externas y asegurar la recepción limpia de paquetes SNMP (puertos UDP `161` y `162`), se ejecutaron las siguientes acciones en Windows 10:

* **Desconexión de Interfaces Virtuales/Externas:** Se desactivó la tarjeta inalámbrica (`Wi-Fi`) y se cerró el cliente VPN (`Tailscale`), eliminando rutas alternativas ajenas a la red del laboratorio.
* **Ajuste de Seguridad Local:** Se desactivó temporalmente el **Firewall de Windows Defender** en perfiles privado y público para permitir el tráfico ICMP y las notificaciones SNMP entrantes.
* **Direccionamiento IPv4 Estático:** Se configuró la tarjeta física `Ethernet` (conectada al puerto 3 del Switch Maipu S3000) con los siguientes parámetros:
  * **Dirección IP:** `192.168.10.20`
  * **Máscara de subred:** `255.255.255.0`
  * **Puerta de enlace predeterminada:** `192.168.10.1`

### Topología Lógica Completa (4 Nodos Activos)

| Dispositivo | Rol en el Laboratorio | Dirección IP | Máscara | Estado ICMP |
| :--- | :--- | :--- | :--- | :--- |
| **Router Maipu MP2804** | Agente SNMP (`ROUTER_GESTIONADO`) | `192.168.10.1` | `255.255.255.0` | Operativo (`TTL=255`, 0% pérdida) |
| **Switch Maipu S3000** | Agente SNMP (`SWITCH_GESTIONADO`) | `192.168.10.2` | `255.255.255.0` | Operativo (`TTL=64`, 0% pérdida) |
| **PC de Escritorio** | Estación Secundaria / Traps | `192.168.10.10` | `255.255.255.0` | Operativo |
| **Portátil ASUS** | Gestor SNMP Principal | `192.168.10.20` | `255.255.255.0` | Operativo |

---

## 2. Configuración y Validación en iReasoning MIB Browser

En la ventana *Advanced Properties of SNMP Agent* de **iReasoning MIB Browser**, se establecieron los parámetros estándar del protocolo:

* **SNMP Version:** `2` (SNMPv2c)
* **Port:** `161`
* **Read Community:** `public`
* **Write Community:** `private`

### Verificación de Identidad de los Agentes (`SNMP GET`)

Se realizó la consulta de lectura sobre el objeto estándar `sysName.0` (OID `.1.3.6.1.2.1.1.5.0`) hacia ambos dispositivos físicos Maipu:

| IP Destino | Puerto | OID Consultado | Objeto MIB | Valor Devuelto | Tipo de Dato | Estado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `192.168.10.2` | `161` | `.1.3.6.1.2.1.1.5.0` | `sysName.0` | `SWITCH_GESTIONADO` | `OctetString` | **Exitoso** |
| `192.168.10.1` | `161` | `.1.3.6.1.2.1.1.5.0` | `sysName.0` | `ROUTER_GESTIONADO` | `OctetString` | **Exitoso** |

---

## 3. Ejecución y Evidencia de la Prueba 1: Flujo de Tráfico en Tiempo Real

### Metodología Aplicada
1. **Inyección de Carga en Red:** Desde la consola CMD de la portátil (`192.168.10.20`), se generó una ráfaga de tráfico ICMP con paquetes de tamaño máximo ($65000\text{ bytes}$) hacia el Router (`192.168.10.1`):
   ```cmd
   ping 192.168.10.1 -t -l 65000
   ```
   * **Resultado de la ráfaga:** 9 paquetes enviados, 9 recibidos (`0% perdidos`), latencia media de $18\text{ ms}$.
2. **Lectura del Contador de Interfaz (`IF-MIB`):** En iReasoning MIB Browser se consultó mediante `SNMP GET` el contador de octetos de entrada de la interfaz troncal del Switch (`Ethernet 1/2`, índice `.2`):
   * **OID:** `.1.3.6.1.2.1.2.2.1.10.2` (`ifInOctets.2`)

### Registro de Variación del Tráfico (`Counter32`)

| Momento de la Consulta | Objeto MIB | Valor Registrado (Bytes) | Incremento Acumulado |
| :--- | :--- | :--- | :--- |
| **Lectura 1 (Reposo inicial)** | `ifInOctets.2` | `64,066` | — |
| **Lectura 2 (Durante ráfaga ICMP)** | `ifInOctets.2` | `600,046` | $+535,980\text{ bytes}$ |
| **Lectura 3 (Final de ráfaga ICMP)** | `ifInOctets.2` | `933,446` | $+869,380\text{ bytes}$ |

---

## 4. Checklist Final para Completar el Laboratorio (Pruebas 2 y 3)

* [x] **Configuración de Agentes Maipu:** Switch S3000 y Router MP2804 operativos y guardados en memoria Flash.
* [x] **Conectividad y Gestor SNMP:** Portátil `192.168.10.20` aislada y comunicando vía SNMPv2c con iReasoning MIB Browser.
* [x] **Prueba 1 (Flujo de Tráfico - GET):** Validada mediante el incremento de `ifInOctets.2` de `64,066` a `933,446` bytes.
* [ ] **Prueba 2 (Detección de Errores - Traps):** Abrir **Tools > Trap Receiver** (`Ctrl + I`) y desconectar por 3 segundos el cable del puerto 2 del Switch para capturar los eventos `linkDown` y `linkUp`.
* [ ] **Prueba 3 (Apagado Remoto - SNMP SET):** Con `ping 192.168.10.1 -t` activo en CMD, enviar `SNMP SET` (`Integer` = `2`) al OID `.1.3.6.1.2.1.2.2.1.7.2` (`ifAdminStatus.2`) en `192.168.10.2` usando la comunidad `private` para apagar el puerto sin usar CLI (y restaurarlo luego con valor `1`).