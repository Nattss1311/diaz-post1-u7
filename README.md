# Post-contenido — Unidad 7: Patrones Arquitectónicos I

**Estudiante:** Natalia Díaz Villamizar  
**Asignatura:** Patrones de Diseño de Software  
**Repositorio:** https://github.com/Nattss1311/diaz-post1-u7

---

## Descripción

Un único proyecto Spring Boot (`multas-biblioteca-api`) para la gestión de multas de una biblioteca universitaria, en dos partes:

1. **Parte 1:** API REST en capas (Model, Repository, Service, Controller) sobre H2, con reglas de negocio para generación de multas, límite de multas pendientes y pago en ventanilla.
2. **Parte 2:** Pago en línea con dos pasarelas intercambiables por configuración (**PagosUDES** y **Wompi**), resuelto con un puerto de dominio y dos adaptadores (arquitectura hexagonal solo en esta porción).

---

## 1. Arquitectura

### Parte 1 — Arquitectura en capas

| Capa | Paquete | Responsabilidad |
| :--- | :--- | :--- |
| **Presentación** | `controller/` | `MultaController` expone `/api/multas`; `GlobalExceptionHandler` traduce excepciones a códigos HTTP. Nunca accede al repositorio: siempre pasa por `MultaService`. |
| **Aplicación** | `service/` | `MultaService` orquesta los casos de uso y aplica el límite de multas pendientes. |
| **Dominio** | `model/` | Entidad `Multa` (con `calcularMonto` y `marcarComoPagada`), `EstadoMulta` y excepciones de negocio. |
| **Infraestructura** | `repository/` | `MultaRepository` extiende `JpaRepository` y agrega `countByEstudianteIdAndEstado`. |

### Parte 2 — Puerto y adaptadores (solo en el pago en línea)

| Paquete | Contenido |
| :--- | :--- |
| `domain/` | `PasarelaPagoPort` (puerto de salida), `ResultadoPago` y `PagoRechazadoException`. Sin imports de Spring ni de clientes HTTP. |
| `infrastructure/pago/` | `PagosUdesAdapter` y `WompiAdapter`: implementan el puerto y traducen cada contrato HTTP externo a `ResultadoPago`. |
| `infrastructure/config/` | `RestTemplateConfig`: bean de `RestTemplate`. |

> `Multa` (en `model/`) es una entidad JPA, por lo que lleva anotaciones de `jakarta.persistence`. La pureza de "sin Spring ni HTTP" aplica a `domain/`, no a `model/`.

---

## 2. Estructura de paquetes

```text
diaz-post1-u7/
├── images/
├── README.md
└── multas-biblioteca-api/
    ├── pom.xml
    └── src/main/
        ├── java/com/example/multas/
        │   ├── MultasApplication.java
        │   ├── controller/
        │   │   ├── MultaController.java
        │   │   └── GlobalExceptionHandler.java
        │   │   └── GenerarMultaRequest.java
        │   ├── service/
        │   │   └── MultaService.java
        │   ├── model/
        │   │   ├── Multa.java
        │   │   ├── EstadoMulta.java
        │   │   ├── MultaNotFoundException.java
        │   │   ├── LimiteMultasPendientesException.java
        │   │   └── MultaYaPagadaException.java
        │   ├── repository/
        │   │   └── MultaRepository.java
        │   ├── domain/
        │   │   ├── port/
        │   │   │   └── PasarelaPagoPort.java
        │   │   ├── ResultadoPago.java
        │   │   └── PagoRechazadoException.java
        │   └── infrastructure/
        │       ├── config/
        │       │   └── RestTemplateConfig.java
        │       └── pago/
        │           ├── PagosUdesAdapter.java
        │           └── WompiAdapter.java
        └── resources/
            └── application.properties
```

> Ajusta el árbol a tu repo real (por ejemplo, dónde está `GenerarMultaRequest`).

---

## 3. Instrucciones de ejecución

### Herramientas utilizadas
- Java 17, Spring Boot 3.x, Spring Data JPA, H2, RestTemplate
- Apache Maven, Thunder Client / Postman, cURL
- Git y GitHub

### Comandos

```bash
git clone https://github.com/Nattss1311/diaz-post1-u7.git
cd diaz-post1-u7/multas-biblioteca-api

# Compilar y empaquetar
mvn clean package

# Ejecutar la aplicación
mvn spring-boot:run
```

La API queda disponible en `http://localhost:8080/api/multas` y la consola H2 en `/h2-console`.

### Selección de la pasarela (`application.properties`)

```properties
# "pagosudes" (por defecto) o "wompi"
app.pagos.proveedor=pagosudes
app.pagos.pagosudes.url=http://localhost:9001/pagosudes/transacciones
app.pagos.wompi.url=http://localhost:9002/wompi/transactions
```

### Endpoints

| Método | Ruta | Descripción |
| :---: | :--- | :--- |
| `GET` | `/api/multas` | Listar multas |
| `GET` | `/api/multas/{id}` | Consultar multa (404 si no existe) |
| `GET` | `/api/multas/estudiante/{estudianteId}` | Multas de un estudiante |
| `POST` | `/api/multas` | Generar multa (201; 400 datos inválidos; 409 límite superado) |
| `PATCH` | `/api/multas/{id}/pagar` | Pago en ventanilla (409 si ya está pagada) |
| `POST` | `/api/multas/{id}/pagar-en-linea` | Pago por la pasarela activa (402 si es rechazado; 409 si ya está pagada) |

---

## 4. Decisiones de diseño

### Punto de decisión 1 — Cálculo del monto: ¿entidad o Service?

El monto (días de atraso × valor por día, con tope) se calcula en el método estático `Multa.calcularMonto`, no en `MultaService`.

**Criterio:** si una regla no necesita ningún colaborador externo (Repository u otro Service) y solo depende de datos que ya recibe como parámetro, es una regla de dominio y vive en el objeto de dominio. **Alternativa descartada:** dejarla como método privado del Service. Funcionaría, pero `Multa` quedaría como un contenedor de datos sin comportamiento (modelo anémico) y cualquier otro punto que necesitara recalcular un monto tendría que duplicar la fórmula o pasar por el Service sin necesitar ninguna de sus dependencias.

### Punto de decisión 2 — Conteo de multas pendientes: ¿consulta o filtrado en memoria?

El límite de multas pendientes (3 por estudiante) se valida con `MultaRepository.countByEstudianteIdAndEstado`, una consulta derivada que Spring Data traduce a un `COUNT` en SQL.

**Justificación:** la decisión de negocio ("¿se le permite una multa más?") la toma `MultaService`, pero el dato que necesita se resuelve donde es eficiente: en el motor de base de datos, que devuelve un único número. **Si el volumen creciera:** la alternativa (`findByEstudianteId` + `stream().filter().count()`) cargaría en memoria todo el historial del estudiante en cada creación de multa, y el tiempo de respuesta se degradaría de forma proporcional al historial; con `COUNT` el costo en la aplicación es constante y la base de datos puede apoyarse en un índice por estudiante y estado.

### Punto de decisión 3 — Selección del adaptador activo

Se usa `@ConditionalOnProperty(prefix = "app.pagos", name = "proveedor")`: solo uno de los dos adaptadores existe en el contexto de Spring, así que `MultaService` pide por constructor un único `PasarelaPagoPort`, sin `@Qualifier` ni condicionales propios.

**Alternativa descartada:** inyectar un `Map<String, PasarelaPagoPort>` y elegir el proveedor en tiempo de ejecución. Es válida y más flexible (permitiría cambiar de pasarela sin reiniciar), pero el requisito real es una pasarela fija por sede durante el piloto, y el `Map` obligaría a `MultaService` a conocer las claves de configuración de cada proveedor. Cambiar de pasarela se hace en `application.properties`, sin recompilar `MultaController` ni `MultaService`.

### Punto de decisión 4 — Diseño del puerto y del tipo de resultado

`PasarelaPagoPort.procesar(Multa)` devuelve `ResultadoPago(proveedor, exitoso, referenciaExterna, mensaje)`. Cada adaptador traduce su formato (`idTransaccion`/`estadoTransaccion` en PagosUDES; `reference`/`status` y montos en centavos en Wompi) a ese mismo tipo.

**Qué se rompería con otro diseño:**
- Si el puerto devolviera el DTO propio de cada pasarela, o tuviera un método por proveedor, `MultaService` tendría que distinguir ambos formatos y agregar una tercera pasarela obligaría a modificarlo.
- Si `ResultadoPago` tuviera un campo `idTransaccion` en lugar de `referenciaExterna`, `WompiAdapter` tendría que forzar un nombre que no describe lo que Wompi devuelve (`reference`): una señal de que el tipo de dominio no sería neutral frente a los proveedores.

### Trade-off considerado (Parte 2)

Se evaluaron tres opciones: **A** (rama `if/switch` en `MultaService`), **B** (interfaz Strategy dentro de `service/`) y **C** (puerto de dominio con adaptadores, elegida).

- **A** es la más rápida de escribir y la de menos clases, pero `MultaService` pasaría a conocer los detalles HTTP de ambas pasarelas y cada proveedor nuevo exigiría modificarlo.
- **B** también resuelve la intercambiabilidad con menos estructura y sin paquete `domain/` separado, y habría sido una decisión válida. Se descartó porque el contrato de salida quedaría dentro de la capa de servicio, junto a Spring, y las diferencias entre los dos formatos HTTP se resolverían allí; con un tercer proveedor o pruebas aisladas con mocks, el puerto en `domain/` los aísla mejor.
- **Lo que se ganó con C:** desacoplamiento del núcleo respecto a las pasarelas, principio abierto/cerrado (agregar un proveedor es agregar un adaptador) y facilidad de prueba.
- **Lo que costó:** dos paquetes nuevos (`domain/`, `infrastructure/`), varias clases adicionales (puerto, resultado, excepción, dos adaptadores y la configuración) y una curva de aprendizaje mayor para un requisito de dos proveedores.
- **Si el piloto terminara y quedara una sola pasarela:** la opción B (o incluso una única clase cliente) bastaría; aun así se conservaría el puerto por la facilidad de probar `MultaService` con mocks y por integraciones futuras.

---

## 5. Conclusiones

La Parte 1 mostró el valor de decidir dónde vive cada regla: el cálculo del monto en la entidad (no necesita colaboradores) y el límite de multas pendientes en el Service apoyado en una consulta agregada, evitando un Service que solo delegue al repositorio. En la Parte 2, el puerto `PasarelaPagoPort` y el tipo neutral `ResultadoPago` permitieron integrar dos pasarelas con contratos HTTP distintos y alternarlas por configuración. Para ello solo se extendió `MultaService` (nueva dependencia y el método `pagarConPasarela`) y `MultaController` (el endpoint `/pagar-en-linea`), sin alterar las reglas de la Parte 1. Lo más difícil de decidir fue si un requisito de solo dos proveedores justificaba el costo de un puerto frente a un Strategy en `service/`; pesó que los contratos HTTP son heterogéneos y que se prevé agregar o quitar proveedores.

---

## 6. Evidencias de pruebas y checkpoints

### Parte 1 — API REST

| Descripción | Imagen |
| :--- | :--- |
| **GET /api/multas (200 OK):** consulta inicial de multas. | ![GET 200](images/checkpoin_get_200OK.png) |
| **POST /api/multas (201 Created):** generación exitosa de multa. | ![POST 201](images/checkpoint_post_JSON_201.png) |
| **POST /api/multas (400 Bad Request):** validación de datos de entrada. | ![POST 400](images/checkpoint_post_400_Bad%20_Request.png) |
| **GET /api/multas/{id} (404 Not Found):** multa inexistente. | ![GET 404](images/checkpoint_status404.png) |
| **POST /api/multas (409 Conflict):** límite de multas pendientes superado. | ![POST 409](images/checkpoint_status409.png) |
| **PATCH /api/multas/{id}/pagar (200 OK):** pago en ventanilla. | ![PATCH 200](images/checkpoint_patch.png) |
| **PATCH /api/multas/{id}/pagar (409 Conflict):** re-pago en ventanilla. | ![PATCH 409](images/checkpoint_patch_status409.png) |

### Parte 2 — Pago en línea (puerto y adaptadores)

| Descripción | Imagen |
| :--- | :--- |
| **PagosUDES (402 Payment Required):** con `app.pagos.proveedor=pagosudes`. | ![PagosUDES 402](images/checkpoint_pagosudes_402.png) |
| **Wompi (402 Payment Required):** con `app.pagos.proveedor=wompi`. | ![Wompi 402](images/checkpoint_wompi402.png) |
| **Independencia del dominio:** `PasarelaPagoPort.java` sin imports de Spring. | ![Domain sin Spring](images/checkpoint_domain_sin_spring.png) |
