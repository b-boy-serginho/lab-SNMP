<?xml version="1.0" encoding="UTF-8"?>
<feedback_laboratorio>
  <metadatos>
    <materia>Redes 2</materia>
    <grupo>Gonzales</grupo>
    <practica>Laboratorio Gestor SNMP</practica>
    <dispositivo_evaluado>Switch Maipu MyPower S3000</dispositivo_evaluado>
    <hostname>SWITCH_GESTIONADO</hostname>
    <estado_general>COMPLETADO_Y_GUARDADO</estado_general>
  </metadatos>

  <evaluacion_tecnica>
    <capa_fisica>
      <puerto_1 interfaz="Ethernet1/1" conexion="PC Gestor SNMP" estado="UP"/>
      <puerto_2 interfaz="Ethernet1/2" conexion="Enlace hacia Router MP2804" estado="UP"/>
    </capa_fisica>

    <direccionamiento_ip>
      <interfaz_gestion>vlan 1</interfaz_gestion>
      <ip>192.168.10.2</ip>
      <mascara>255.255.255.0</mascara>
    </direccionamiento_ip>

    <configuracion_snmp>
      <servicio estado="habilitado" comando="snmp-server enable"/>
      <comunidad_lectura nombre="public" permiso="ro" comando="snmp-server community ro public"/>
      <comunidad_escritura nombre="private" permiso="rw" comando="snmp-server community rw private"/>
      <destino_traps ip="192.168.10.10" version="v2c" comunidad="public" comando="snmp-server host 192.168.10.10 v2c public"/>
      <persistencia estado="OK" comando="write"/>
    </configuracion_snmp>
  </evaluacion_tecnica>

  <observaciones_y_recomendaciones>
    <observacion tipo="fortaleza">
      Uso efectivo de la ayuda contextual (?) en CLI para adaptar la sintaxis de MyPowerOS respecto a la guia generica.
    </observacion>
    <observacion tipo="precaucion_prueba_set">
      Al ejecutar el comando SNMP SET sobre ifAdminStatus (.1.3.6.1.2.1.2.2.1.7), seleccionar el indice del puerto Ethernet1/2 (hacia el router) y no el Ethernet1/1 para evitar perder conexion con el gestor.
    </observacion>
    <observacion tipo="red_host">
      Verificar que el Firewall de Windows en 192.168.10.10 tenga abiertos los puertos UDP 161 y 162 para recibir traps linkDown/linkUp.
    </observacion>
  </observaciones_y_recomendaciones>

  <siguientes_pasos>
    <paso numero="1">Conectar puerto 2 del Switch al puerto FE0 del Router Maipu MP2804.</paso>
    <paso numero="2">Configurar IP 192.168.10.1/24 y Agente SNMP en el Router MP2804.</paso>
    <paso numero="3">Verificar conectividad ping desde 192.168.10.10 hacia ambos agentes.</paso>
  </siguientes_pasos>
</feedback_laboratorio>