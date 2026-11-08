# Validación documental y límites

[← README](../README.md)

## Realizado

- Se recalcularon MD5, SHA-1 y SHA-256 de las tres muestras del caso B y se contrastaron sus SHA-256 de referencia.
- Se comprobaron las versiones locales documentadas de Python, Capstone, pyelftools y PyCryptodomex.
- Se revisaron enlaces/anclas locales, estructura de CSV, separación de casos, estado de los pendientes y manifiesto SHA-256.
- Se conservan resultados documentados de las revisiones previas: 38 instrucciones y 11 reglas de estado para autenticación/contadores, además de 72 comprobaciones matemáticas de cifrado con claves sintéticas. Son comprobaciones del modelo y del recorrido revisado, no una ejecución del malware ni una certificación integral.

## Alcance de esta entrega

El paquete contiene documentación y datos de análisis. No se distribuyen herramientas, scripts, reglas YARA/Suricata, clientes de red ni muestras. Las menciones a herramientas en la metodología describen cómo se obtuvieron los resultados; no indican que se incluyan implementaciones.

## Pendiente para una futura detección operativa

- Diseñar reglas que distingan hash exacto, combinación de rasgos y señales de comportamiento.
- Compilar y probar con las versiones reales de YARA/Suricata y la configuración del sensor.
- Evaluar positivos autorizados y negativos representativos de firmware limpio.
- Correlacionar el dominio DNS con QTYPE TXT; no esperar visibilidad del nombre dentro de DoH cifrado.
- Medir ruido de servicios públicos, NAT y puertos aislados antes de considerar bloqueos.

No se realizó compilación con motores YARA/Suricata ni evaluación en producción. No hay corpus amplio limpio ni PCAP de sesión disponibles. No se promete una firma universal del protocolo de `u`.
