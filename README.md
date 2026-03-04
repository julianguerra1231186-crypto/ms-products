# API Gateway - PulpApp
## Descripción

El **API Gateway** es el componente encargado de centralizar y gestionar las solicitudes de los clientes hacia los microservicios del sistema **PulpApp - Sistema Distribuido de Venta Online de Pulpas Naturales**.
Este gateway actúa como un punto de entrada único para el sistema, permitiendo redirigir las peticiones hacia los servicios correspondientes.
En el caso de este proyecto, el gateway se encarga de enrutar las solicitudes hacia el microservicio **ms-users**, responsable de la gestión de usuarios.
---
# Arquitectura
Dentro de la arquitectura del sistema distribuido, el API Gateway cumple las siguientes funciones:

* Punto de entrada único para el sistema
* Enrutamiento hacia microservicios
* Simplificación del acceso desde el frontend
* Organización de las rutas de la API
* 
Arquitectura simplificada:

```
Frontend
   |
   v
API Gateway
   |
   v
Microservicio ms-users
   |
   v
PostgreSQL
```
---
# Tecnologías utilizadas

El API Gateway fue desarrollado utilizando las siguientes tecnologías:

* Java 17
* Spring Boot
* Spring Cloud Gateway
* Maven
* Docker
--
# Estructura del proyecto
```
api-gateway
│
├── src
│   └── main
│       ├── java
│       │   └── com.pulpapp.gateway
│       │       └── ApiGatewayApplication.java
│       │
│       └── resources
│           └── application.yml
│
└── pom.xml
```
---

# Configuración del Gateway
El gateway utiliza **Spring Cloud Gateway** para redirigir las solicitudes hacia los microservicios.
Ejemplo de configuración en `application.yml`:

```
spring:
  cloud:
    gateway:
      routes:
        - id: ms-users
          uri: http://ms_users_service:8081
          predicates:
            - Path=/api/users/**
```

Esto significa que todas las solicitudes que comiencen con:
```
/api/users
```
serán redirigidas al microservicio:
```
ms-users
``
--
# Ejecución del Gateway
Para ejecutar el API Gateway se deben seguir los siguientes pasos:
## 1 Compilar el proyecto
```
mvn clean package
```
---
## 2 Ejecutar con Docker
Desde la raíz del proyecto:
```
docker compose up --build
``
--
# Pruebas del Gateway
Una vez ejecutado el gateway, se pueden realizar pruebas desde **Postman** o el navegador.
```
http://localhost:8080/api/users
```
El gateway redirigirá automáticamente la petición al microservicio **ms-users**.
---
# Beneficios del API Gateway
El uso de un API Gateway permite:
* Centralizar el acceso a los microservicios
* Simplificar las rutas de acceso
* Escalar servicios de forma independiente
* Mejorar la organización de la arquitectura distribuida
---
# Autor
Proyecto desarrollado como parte de la asignatura **Sistemas Distribuidos**.
Autor:
Julian Guerra
