# Feedback Técnico Integral — Switch Maipu S3000 y Router Maipu MP2804

**Materia:** Redes 2  
**Docente:** Gonzales  
**Práctica:** Laboratorio Gestor SNMP  
**Estado Actual:** Infraestructura de Red y Agentes SNMP 100% Operativos (Listo para Fase de Software Gestor)

---

## 1. Resumen Ejecutivo
Has completado de manera exitosa y verificada la implementación física, lógica y de agentes SNMP en ambos equipos físicos **Maipu** (`SWITCH_GESTIONADO` y `ROUTER_GESTIONADO`), así como la conectividad bidireccional completa desde la estación de administración (`192.168.10.10`).

* **Nivel de avance del laboratorio:** **75% completado** (Capa física, direccionamiento IP, agentes SNMP RO/RW, envío de Traps y pruebas ICMP al 100%).
* **Fase restante:** Instalación/configuración del software Gestor SNMP Real en Windows, carga de las MIBs de Maipu y ejecución de las pruebas en vivo (Flujo, Errores y Apagado por `SNMP SET`).

---

## 2. Evaluación por Dispositivo

### A. Switch Maipu MyPower S3000 (`SWITCH_GESTIONADO`)
* **Topología y Enlaces Físicos:**
  * Puerto `Ethernet 1/1`: Conectado a la estación de gestión (PC/Laptop `192.168.10.10`) $\rightarrow$ Estado `UP`.
  * Puerto `Ethernet 1/2`: Conectado al puerto `FE0` del Router MP2804 (`192.168.10.1`) $\rightarrow$ Estado `UP`.
* **Direccionamiento de Administración:**
  * Interfaz virtual `vlan 1` configurada con `192.168.10.2 255.255.255.0` y levantada con `no shutdown`.
* **Agente SNMP Aplicado (Sintaxis MyPowerOS S3000):**
  * Servicio habilitado: `snmp-server enable`
  * Comunidad de lectura (`GET`): `snmp-server community ro public`
  * Comunidad de lectura/escritura (`SET`): `snmp-server community rw private`
  * Destino de notificaciones (`Traps`): `snmp-server host 192.168.10.10 v2c public`
  * Guardado en Flash: `write` (`Write configuration successfully!`).

### B. Router Maipu MP2804 (`ROUTER_GESTIONADO`)
* **Limpieza y Direccionamiento IP:**
  * Se reemplazó la configuración previa (`R3` / `10.0.0.3`) asignando a `fastethernet 0` (`FE0`) la IP `192.168.10.1 255.255.255.0` en estado `UP`, y apagando administrativamente `fastethernet 1` (`shutdown`).
* **Agente SNMP Aplicado (Sintaxis MP2804):**
  * Servicio SNMP activo (`snmp-server start`) y traps habilitados (`snmp-server enable traps snmp linkdown linkup`).
  * Vista completa de MIB definida (`snmp-server view fullview 1.3.6.1 include`) y vinculada a la comunidad de escritura: `snmp-server community private view fullview rw`.
  * Comunidad de lectura activa: `snmp-server community public view readonlyview ro`.
  * Destino de notificaciones (`Traps`): `snmp-server host 192.168.10.10 traps community public`.
  * Guardado en Flash: `write` (`Write to startup file ... OK`).

### C. Verificación de Conectividad (Capa 3)
* **Desde el Router (`192.168.10.1`) hacia el Switch (`192.168.10.2`):** `Success rate is 100% (5/5)`.
* **Desde la PC Gestora (`192.168.10.10`) hacia el Switch (`192.168.10.2`):** 4 enviados, 4 recibidos (`0% perdidos`, `TTL=64`).
* **Desde la PC Gestora (`192.168.10.10`) hacia el Router (`192.168.10.1`):** 4 enviados, 4 recibidos (`0% perdidos`, `TTL=255`).

---

## 3. Fortalezas Demostradas y Observaciones para el Informe

1. **Adaptación de Sintaxis entre Modelos Maipu (Punto Fuerte para el Informe):**
   * Lograste resolver mediante ayuda contextual (`?`) que, aunque ambos equipos son de la marca Maipu, utilizan sintaxis distintas en sus firmwares:
     * El **Switch S3000** exige el permiso antes de la comunidad (`snmp-server community rw private`) y `v2c` en el host (`snmp-server host 192.168.10.10 v2c public`).
     * El **Router MP2804** exige una vista MIB asociada (`snmp-server community private view fullview rw`) y la estructura `traps community` en el host (`snmp-server host 192.168.10.10 traps community public`).
   * *Recomendación:* Documenta esta comparativa en tu informe escrito; demuestra dominio real sobre hardware físico frente a configuraciones genéricas de simulador.

2. **Limpieza Opcional en el Switch (`10.0.0.10`):**
   * En el `show running-config` del Switch quedó una línea residual de fábrica (`snmp-server host 10.0.0.10 v2c public`). No afecta el funcionamiento, pero si deseas dejar la configuración limpia para las capturas del informe, puedes eliminarla con `no snmp-server host 10.0.0.10 v2c public` y guardar con `write`.

3. **Firewall de Windows para Traps (UDP 162):**
   * Mantén desactivado temporalmente el Firewall de Windows en la PC `192.168.10.10` (o crea una regla de entrada para UDP 161 y 162) durante la práctica y la defensa del viernes para garantizar que el gestor reciba los traps `linkDown` y `linkUp` enviados por ambos equipos.

---

## 4. Checklist Final para Completar el 100% del Laboratorio
- [ ] **Paso 1:** Abrir el Gestor SNMP Real en Windows (`192.168.10.10`), importar las MIBs de Maipu + `IF-MIB` y agregar `192.168.10.1` y `192.168.10.2` con SNMPv2c (`public` / `private`).
- [ ] **Paso 2 (Flujo):** Ejecutar `ping 192.168.10.1 -t -l 65000` desde Windows y capturar la gráfica de tráfico en bps subiendo en el gestor.
- [ ] **Paso 3 (Errores/Traps):** Desconectar brevemente el cable del puerto 2 del Switch y capturar la alerta `linkDown` / `linkUp` en el gestor.
- [ ] **Paso 4 (Prueba SET):** Con `ping 192.168.10.1 -t` corriendo, enviar un `SNMP SET` con comunidad `private` sobre `ifAdminStatus` (`.1.3.6.1.2.1.2.2.1.7.2`) con valor entero **`2`** (`down`), verificando el apagado físico del puerto 2 en el switch y el corte inmediato del ping sin tocar la consola CLI.