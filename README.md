# Universidad de las Americas
## Integrantes:
Sebastian Abad
Adrian Cornejo
Joseph Flores

# Enrollments API — Banner

## 1. Problema de integración

### Contexto

En la universidad conviven tres sistemas que participan en el proceso de matrícula, cada uno con una responsabilidad distinta:

- **Banner:** es el sistema de información académica de la institución y la fuente oficial de los datos académicos. Gestiona estudiantes, carreras, periodos académicos, oferta de materias con sus cupos y prerrequisitos, y el registro de las matrículas.
- **Brightspace:** es el sistema de gestión del aprendizaje. En él se crean las aulas virtuales de cada materia, donde los estudiantes acceden a contenidos, entregan tareas y rinden evaluaciones. Para habilitar a un estudiante en un aula, necesita saber en qué materias está matriculado.
- **Sistema Financiero:** gestiona la facturación y los cobros de la universidad. Emite el valor de la matrícula según la carrera del estudiante y registra su pago, por lo que necesita saber qué matrículas existen y en qué estado se encuentran.

Cada sistema tiene su propia base de datos y su propia tecnología. Brightspace y el Sistema Financiero dependen de información que solo Banner posee, pero ninguno debería acceder directamente a la base de datos de Banner.

### Propósito

La Enrollments API expone las matrículas registradas en Banner para que otros sistemas institucionales las consulten y gestionen en tiempo real. Sin ella, Brightspace no sabría qué estudiantes habilitar en cada aula virtual al inicio del periodo, y el Sistema Financiero trabajaría con información desactualizada o tendría que obtenerla de forma manual. La API sustituye esos procesos manuales por un contrato único y estable entre los sistemas.

## 2. Sistemas involucrados

| Rol | Sistema | Uso de la API |
|---|---|---|
| Proveedor | Banner (sistema académico) | Registra y administra las matrículas |
| Consumidor | Brightspace (LMS) | Habilita a los estudiantes matriculados en sus aulas virtuales |
| Consumidor | Sistema Financiero | Factura la matrícula según la carrera |

## 3. Recursos principales

| Recurso | Descripción | Ejemplo de ID |
|---|---|---|
| `Enrollment` | Matrícula de un estudiante en un periodo, con su lista de materias | `E-2026-00123` |
| `Student` | Estudiante | `S-10245` |
| `Program` | Carrera (pregrado/posgrado) | `P-ISW` |
| `AcademicTerm` | Periodo académico semestral | `2026-2` |
| `Course` | Materia | `ISWZ3104` |

## 4. Endpoints

| Método | Endpoint | Descripción |
|---|---|---|
| POST | `/enrollments` | Crear una matrícula |
| GET | `/enrollments/{enrollmentId}` | Consultar una matrícula |
| GET | `/enrollments?studentId=&termId=&status=` | Buscar matrículas |
| PATCH | `/enrollments/{enrollmentId}` | Modificar parcialmente una matrícula (estado, materias) |

### Respuestas por endpoint

| Endpoint | Éxito | Errores |
|---|---|---|
| `POST /enrollments` | 201 | 400 `VALIDATION_ERROR`, 409 `ALREADY_ENROLLED`, 500 |
| `GET /enrollments/{enrollmentId}` | 200 | 404 `ENROLLMENT_NOT_FOUND`, 500 |
| `GET /enrollments` | 200 (lista, puede ser `[]`) | 400 `VALIDATION_ERROR`, 500 |
| `PATCH /enrollments/{enrollmentId}` | 200 | 404 `ENROLLMENT_NOT_FOUND`, 422 `COURSE_FULL` / `INVALID_STATUS_TRANSITION`, 500 |

Estados de una matrícula: `PENDING` (registrada, pendiente de pago) → `CONFIRMED` (pagada) o `CANCELLED` (anulada).

## 5. Códigos HTTP utilizados

| Código | Nombre | Cuándo se usa | Ejemplo en esta API |
|---|---|---|---|
| 200 | OK | La operación fue exitosa y se devuelve información | GET de una matrícula, búsqueda, PATCH |
| 201 | Created | Se creó un recurso nuevo | POST `/enrollments` |
| 204 | No Content | Éxito sin nada que devolver | No se usa (no hay DELETE) |
| 400 | Bad Request | La solicitud está mal formada: falta un campo obligatorio, tipo incorrecto o valor fuera del enum | POST sin `studentId`; búsqueda con `status=PAGADO` |
| 404 | Not Found | El recurso indicado en la ruta no existe | GET o PATCH a `/enrollments/E-9999` inexistente |
| 409 | Conflict | La solicitud choca con el estado actual de otro recurso | El estudiante ya tiene matrícula en ese periodo |
| 422 | Unprocessable Entity | La solicitud está bien formada pero viola una regla de negocio | Materia sin cupo, prerrequisito faltante, pasar de `CANCELLED` a `CONFIRMED` |
| 500 | Internal Server Error | Falla inesperada del proveedor; el consumidor no hizo nada mal | Banner no puede conectarse a su base de datos |

**Regla para decidir:**
- ¿El JSON o los parámetros están mal escritos? → **400**
- ¿El recurso de la URL no existe? → **404**
- ¿Ya existe algo que lo impide? → **409**
- ¿Todo está bien escrito, pero el negocio no lo permite? → **422**
- ¿Falló Banner por dentro? → **500**

> Una búsqueda sin resultados **no es 404**: devuelve **200** con una lista vacía `[]`. La colección `/enrollments` sí existe; solo no hay elementos que cumplan el filtro.

## 6. Modelo de errores

Todos los errores usan el schema `Error`:

```json
{
  "code": "COURSE_FULL",
  "message": "Course ISWZ3104 has no available seats",
  "correlationId": "ab23f8b1-91c2-45dc"
}
```

- `code`: identificador fijo para que el consumidor reaccione programáticamente.
- `message`: explicación legible del caso concreto.
- `correlationId`: identifica la petición para rastrearla en los logs de Banner.

| HTTP | code |
|---|---|
| 400 | `VALIDATION_ERROR` |
| 404 | `ENROLLMENT_NOT_FOUND` |
| 409 | `ALREADY_ENROLLED` |
| 422 | `COURSE_FULL`, `INVALID_STATUS_TRANSITION` |
| 500 | `INTERNAL_ERROR` |

## 7. Decisiones de diseño

1. **La matrícula se modela por periodo y contiene la lista de materias.** El Sistema Financiero factura por carrera y por matrícula completa, no por materia. Brightspace obtiene los códigos de las materias desde la misma respuesta, sin hacer llamadas adicionales.
2. **Los filtros de búsqueda son query parameters de una sola colección.** Se usa `GET /enrollments?termId=&studentId=&status=` en lugar de rutas como `/enrollments/byTerm`. Los filtros son opcionales y combinables, y agregar uno nuevo no obliga a crear otra ruta.
3. **La modificación usa PATCH y no PUT.** Los cambios habituales son parciales, como confirmar el pago (`status`) o retirar una materia (`courses`). Con PUT, el consumidor tendría que reenviar la matrícula completa y podría sobrescribir datos que no conoce.
4. **El consumidor no envía los datos que genera el proveedor.** `id`, `status` inicial, `createdAt` y `updatedAt` los asigna Banner. Por eso el request (`EnrollmentCreate`) y el response (`Enrollment`) son schemas distintos. El response devuelve siempre el recurso completo.
5. **Se diferencia 400 de 422.** 400 indica una solicitud mal formada (campo faltante o valor fuera del enum) y se agrupa bajo un único `VALIDATION_ERROR`. 422 indica una solicitud válida que viola una regla de negocio, como la falta de cupo. Así el consumidor sabe si debe corregir su integración o informar al usuario.
6. **Una búsqueda sin resultados devuelve 200 con `[]`.** La colección existe; que no haya coincidencias no es un error.
7. **Los modelos se reutilizan.** Los schemas están en `components/schemas`, y el error 500 en `components/responses/InternalError`, referenciados con `$ref` desde todas las operaciones.

## 8. Archivos y prueba

```
tallerAPI/
├── openapi.yaml                       # contrato OpenAPI 3.0.3
├── README.md
└── postman/
    └── API.postman_collection.json    # colección generada desde openapi.yaml
```

La colección se importa en Postman (Import → `postman/API.postman_collection.json`) y contiene 4 requests preparados con los valores de ejemplo del contrato: POST, GET por ID, búsqueda por `studentId`/`termId`/`status` y PATCH. Como no hay backend, cada request incluye como *examples* las respuestas definidas en el contrato.

## 9. Reflexión final

**¿Qué ventaja obtiene el sistema consumidor al disponer de un contrato OpenAPI antes de que el proveedor haya terminado de implementar la API?**

Puede desarrollar su parte en paralelo en vez de esperar a Banner. El equipo de Brightspace ya sabe qué URL llamar, qué JSON enviar, qué recibirá y qué errores debe manejar. Por ejemplo, puede programar desde hoy qué hacer ante un `422 COURSE_FULL` o un `404 ENROLLMENT_NOT_FOUND`, y probar contra un mock generado desde el mismo `openapi.yaml`. Además, los desacuerdos se detectan cuando cambiar el diseño todavía es barato: si al Sistema Financiero le faltara `programId` en la respuesta, se corrige en el YAML, no en dos sistemas ya implementados.

Lo que el consumidor **necesita conocer** es solo el contrato: recursos, operaciones, formatos, estados y códigos de error. Lo que **puede ignorar** es todo lo interno de Banner: su base de datos, su lenguaje, cómo calcula los cupos o cómo valida los prerrequisitos. Mientras el contrato se respete, Banner puede cambiar su implementación sin romper a Brightspace ni al Sistema Financiero.
