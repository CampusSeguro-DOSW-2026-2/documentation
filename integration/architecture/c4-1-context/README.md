# C4 nivel 1: contexto del sistema

Esta carpeta guarda el diagrama de **contexto** de CampusSeguro: el sistema completo como una sola caja, las personas que lo usan y los sistemas externos con los que se comunica.

## Qué debe mostrar el diagrama

* **Personas:** los usuarios de la comunidad del campus según los roles iniciales de la [matriz de roles y permisos](../../permissions-matrix.md) (estudiante, profesor, administrativo, brigadista, coordinador de emergencia, organizador de eventos, administrador y auditor).
* **El sistema CampusSeguro** como una sola caja, sin mostrar el gateway ni los microservicios.
* **Sistemas externos:** el servicio de correo electrónico que usan MS01 y MS03 para enviar mensajes y el proveedor del modelo de lenguaje que consulta el tutor de MS07.
* **Relaciones:** qué hace cada persona con el sistema y qué envía el sistema a cada sistema externo.

## Archivo esperado

`c4-1-context.drawio.svg`, editable en draw.io. Los nombres de los diagramas siguen la convención de [CONTRIBUTING.md](../../../CONTRIBUTING.md).

Este archivo se elimina cuando se agrega el diagrama, como lo describe CONTRIBUTING.md.
