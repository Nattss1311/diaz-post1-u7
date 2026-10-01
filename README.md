# Sistema de Gestión de Multas - Biblioteca API (Parte 1)

**Estudiante:** Natalia Díaz Villamizar  
**Asignatura:** Patrones de Diseño / Ingeniería de Software  
**Repositorio:** https://github.com/Nattss1311/diaz-post1-u7.git

---

## 1. Descripción de la Arquitectura en Capas

El proyecto está construido bajo la arquitectura en capas estándar de Spring Boot, garantizando la separación de responsabilidades:

1. **`model` (Capa de Dominio):** Contiene la entidad JPA `Multa` y el enum `EstadoMulta`. Es la encargada de representar las tablas de la base de datos y albergar lógica propia de la entidad (como el método estático `Multa.calcularMonto`).
2. **`repository` (Capa de Acceso a Datos):** Interfaz `MultaRepository` que extiende de `JpaRepository`. Define consultas personalizadas como `countByEstudianteIdAndEstado`.
3. **`service` (Capa de Negocio):** Clase `MultaService` que orquesta la lógica de negocio, validando reglas (límite de multas pendientes) e interactuando con la capa de persistencia.
4. **`controller` (Capa de Presentación / API REST):** Expone los endpoints HTTP con `MultaController` y gestiona las respuestas o excepciones globales con `GlobalExceptionHandler`.

---

## 2. Diagrama de Estructura de Paquetes
```bash 
diaz-post1-u7/
└──images/
└── multas-biblioteca-api/
├── pom.xml
└── src/main/java/com/example/multas/
├── controller/
│   ├── MultaController.java
│   └── GlobalExceptionHandler.java
├── model/
│   ├── Multa.java
│   ├── EstadoMulta.java
│   ├── GenerarMultaRequest.java
│   └── PagarMultaRequest.java
├── repository/
│   └── MultaRepository.java
└── service/
├── MultaService.java
└── exception/
├── MultaNotFoundException.java
├── MultaYaPagadaException.java
└── LimiteMultasPendientesException.java
```
---

## 3. Instrucciones de Ejecución

1. Clonar el repositorio:
   ```bash
   git clone [https://github.com/Nattss1311/diaz-post1-u7.git](https://github.com/Nattss1311/diaz-post1-u7.git)
   cd diaz-post1-u7/multas-biblioteca-api

2. Compilar el proyecto:

```Bash
mvn clean compile
```

3. Ejecutar la aplicación:

```Bash
mvn spring-boot:run
```
**La API quedará escuchando en http://localhost:8080/api/multas.**

## 4. Decisiones de Arquitectura y Diseño
### Punto de Decisión 1: Ubicación del Cálculo del Monto de la Multa

El cálculo del monto de la multa (días de atraso × valor por día) no requiere de colaboradores o datos externos: depende únicamente de los días de retraso recibidos como parámetro. Se optó por ubicar esta regla como un método estático de la propia entidad `Multa` `(Multa.calcularMonto)` , en lugar de alojarla como un método privado dentro de `MultaService`.
**Justificación:** Se busca evitar el antipatrón de Modelo Anémico, en el cual las entidades actúan como simples contenedores de datos sin comportamiento propio. Si la regla no necesita acceso a un repositorio u otro servicio, la responsabilidad lógica pertenece al dominio. Esto permite que cualquier otro componente que requiera calcular un monto pueda reusar la regla directamente desde la entidad sin obligarlo a pasar por la capa de servicio.

### Punto de Decisión 2: Conteo de Multas Pendientes a Nivel de Base de Datos

Para validar si a un estudiante se le permite generar una nueva multa (límite de 3 multas pendientes), se implementó la consulta derivada `MultaRepository.countByEstudianteIdAndEstado` en lugar de traer todas las multas del estudiante a memoria con `findByEstudianteId` y filtrarlas mediante Streams de Java.

**Justificación:** La decisión de negocio la toma `MultaService`, pero la agregación de datos se delega al motor relacional, que ejecuta la consulta `COUNT` en SQL de manera directa y eficiente.Impacto en Escalabilidad: Cargar el historial completo de multas en memoria para realizar el conteo causaría una degradación en el tiempo de respuesta proporcional al volumen de registros del estudiante. Resolver la consulta directamente en la base de datos garantiza un rendimiento constante ($O(1)$ a nivel de aplicación) independientemente del tamaño del historial.

