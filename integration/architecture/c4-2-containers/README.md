# C4 nivel 2: contenedores

Esta carpeta guarda el diagrama de **contenedores** de CampusSeguro: las aplicaciones y los almacenes de datos que forman el sistema y cómo se comunican entre sí.

## Qué debe mostrar el diagrama

* **Frontend:** aplicación web estática que consume el API Gateway.
* **API Gateway:** puerta única de todo el tráfico, tanto externo como entre microservicios.
* **Microservicios MS01 a MS08**, cada uno con su base de datos propia.
* **Sistemas externos** del diagrama de contexto, conectados al microservicio que los usa.
* **Relaciones y protocolos:**
  * frontend → gateway por HTTPS con JWT;
  * gateway → microservicios por HTTPS con mTLS en la nube;
  * microservicio → gateway → microservicio con la clave de servicio en las llamadas internas (las llamadas registradas están en los [Estándares §5](../../standards.md));
  * gateway → MS01 para consultar los cortes de sesión;
  * microservicios → MS08 para los eventos de auditoría.

Los puertos y el aislamiento de red están en los [Estándares §2](../../standards.md) y en la [conexión al API Gateway](../../gateway-connection.md).

## Archivo esperado

`c4-2-containers.drawio.svg`, editable en draw.io. Los nombres de los diagramas siguen la convención de [CONTRIBUTING.md](../../../CONTRIBUTING.md).

Este archivo se elimina cuando se agrega el diagrama, como lo describe CONTRIBUTING.md.
