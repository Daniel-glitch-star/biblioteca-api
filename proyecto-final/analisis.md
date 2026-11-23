# Fase 1. Análisis del problema

## 1. Descripción del problema

La API de la biblioteca actualmente permite registrar libros, consultar su disponibilidad y gestionar los préstamos de los usuarios. El nuevo requerimiento consiste en agregar el préstamo de laptops, conservando las funciones que ya existen.

El principal problema es que el sistema de préstamos fue desarrollado pensando únicamente en libros. Por ejemplo, cada préstamo debe tener un `book_id` y el inventario se controla mediante la cantidad de copias disponibles. Esto funciona para los libros, pero no se puede utilizar directamente para una laptop, que puede necesitar identificarse por su número de serie y manejarse como un equipo individual.

Por eso, antes de modificar el código es necesario revisar qué partes dependen de los libros y comparar distintas formas de agregar el nuevo recurso. La solución debe permitir prestar y devolver laptops sin afectar los préstamos actuales.

## 2. Revisión de la estructura actual

El proyecto está desarrollado en Go, utiliza Gin para atender las solicitudes y MySQL para almacenar la información. Su organización sigue una arquitectura por capas: el *handler* recibe las peticiones, el *service* aplica las reglas de negocio y el *repository* realiza las consultas a la base de datos.

Los archivos revisados del módulo de préstamos son los siguientes:

| Archivo | Función dentro del sistema |
| --- | --- |
| `internal/handler/routes.go` | Registra las rutas de préstamos y aplica la autenticación con JWT. |
| `internal/handler/loan_handler.go` | Recibe las solicitudes para prestar, devolver y consultar préstamos. |
| `internal/service/loan_service.go` | Comprueba la disponibilidad, evita que un usuario pida dos veces el mismo libro y calcula las fechas de devolución. |
| `internal/repository/loan_repository.go` | Guarda y consulta los préstamos en la base de datos. |
| `internal/repository/book_repository.go` | Consulta los libros y aumenta o disminuye sus copias disponibles. |
| `internal/models/loan.go` | Define los datos de los préstamos y de las solicitudes. |
| `internal/models/book.go` | Define los datos del inventario de libros. |
| `migrations/001_init.sql` | Crea las tablas `users`, `books` y `loans`, con sus relaciones. |
| `cmd/api/main.go` | Conecta los repositorios, servicios y handlers para iniciar la API. |

## 3. Funcionamiento de un préstamo de libro

El proceso comienza cuando un usuario autenticado hace una solicitud `POST /api/loans` con el identificador del libro (`book_id`). La ruta pasa por el middleware de autenticación, que verifica el token y permite identificar al usuario.

Después, `loan_handler.go` recibe los datos de la solicitud y los envía a `loan_service.go`. Allí se consulta si el libro existe y si tiene copias disponibles. También se verifica si el usuario ya tiene un préstamo activo de ese mismo libro. Si se cumplen las condiciones, el servicio calcula una fecha de vencimiento de 14 días.

Finalmente, `loan_repository.go` registra el préstamo en la tabla `loans` y `book_repository.go` descuenta una copia del inventario. Al devolver el libro mediante `PUT /api/loans/:id/return`, el servicio verifica que el préstamo exista y que no haya sido devuelto; luego lo marca como devuelto y aumenta nuevamente las copias disponibles.

En la versión revisada, el registro del préstamo y el descuento del inventario se realizan en operaciones separadas. Si una operación funciona y la otra falla, podría quedar una diferencia entre los préstamos registrados y las copias disponibles. Es un aspecto que conviene tener en cuenta al diseñar el módulo de laptops.

## 4. Puntos de acoplamiento con los libros

Se identificaron varias partes del código que asumen que el único recurso prestable es un libro:

| Ubicación | Dependencia identificada | Consecuencia al agregar laptops |
| --- | --- | --- |
| `migrations/001_init.sql` | La tabla `loans` tiene un `book_id` obligatorio relacionado con `books`. | No es posible registrar allí una laptop sin cambiar el esquema. |
| `internal/models/loan.go` | `Loan` y `LoanRequest` utilizan `BookID`; `LoanDetail` incluye título y autor del libro. | Los datos del préstamo no representan directamente las características de una laptop. |
| `internal/service/loan_service.go` | El servicio consulta el repositorio de libros, verifica `AvailableCopies` y utiliza `BookID`. | Las reglas de préstamo dependen del inventario de libros. |
| `internal/repository/loan_repository.go` | Las consultas guardan `book_id` y usan `JOIN books` para mostrar los préstamos. | Las consultas no permiten mostrar laptops sin cambios. |
| `internal/repository/book_repository.go` | La disponibilidad se modifica aumentando o disminuyendo `available_copies`. | El inventario está pensado para varias copias de un libro y no para equipos individuales. |
| `internal/handler/loan_handler.go` | El handler recibe `LoanRequest` y devuelve respuestas asociadas a los préstamos de libros. | Sería necesario adaptar las solicitudes o crear otras para laptops. |
| `internal/handler/routes.go` | Las rutas `/api/loans` están conectadas con el handler actual de préstamos. | El nuevo flujo necesita rutas propias o cambios en las existentes. |
| `cmd/api/main.go` | El servicio de préstamos se construye con `LoanRepository` y `BookRepository`. | Una solución compartida requeriría modificar cómo se conectan los componentes. |

Estos puntos muestran que el sistema no está preparado para incorporar nuevos tipos de recursos sin hacer modificaciones. Si más adelante la biblioteca presta tablets, proyectores u otros equipos, volvería a aparecer el mismo problema.

## 5. Alternativas de diseño

### Opción 1. Crear un módulo independiente para laptops

Consiste en agregar tablas para las laptops y sus préstamos, junto con sus propios modelos, repositorios, servicios, handlers y rutas. El módulo de libros seguiría funcionando de la misma manera y ambos módulos utilizarían el sistema de usuarios y autenticación existente.

Su principal ventaja es que requiere pocos cambios en el código actual, por lo que disminuye el riesgo de afectar los préstamos de libros. También permite definir reglas específicas para las laptops, como la identificación por número de serie y el control de disponibilidad individual.

Como desventaja, se repetiría parte de la lógica de préstamos. Si posteriormente se agregan proyectores o tablets, probablemente habría que crear nuevos módulos con funciones similares, aumentando el trabajo de mantenimiento.

### Opción 2. Crear un sistema de préstamos para diferentes recursos

Esta alternativa consiste en utilizar una estructura más general, donde los libros y las laptops sean tipos de recursos que se pueden prestar. Se necesitarían cambios en la base de datos, los modelos y los servicios para que los préstamos no dependan directamente de `book_id`.

La ventaja es que facilitaría incorporar otros recursos en el futuro, porque todos utilizarían una base común para registrar y consultar préstamos. También podría evitar que se repitan algunas funciones.

La principal desventaja es que exige modificar una parte importante del sistema existente. Esto aumenta la complejidad y el riesgo de introducir errores en funciones que ya estaban funcionando. Además, habría que tener cuidado con las diferencias entre prestar varias copias de un libro y prestar un equipo específico.

### Opción 3. Ampliar el préstamo actual para aceptar libros o laptops

En este caso se mantendría el módulo de préstamos existente, pero se agregarían campos y condiciones para distinguir si la solicitud corresponde a un libro o a una laptop. La base de datos y las consultas tendrían que modificarse para guardar ambas posibilidades.

Esta opción permitiría reutilizar parte del código que ya existe y evitaría construir un módulo de préstamos completamente separado. Sin embargo, el servicio tendría que manejar reglas diferentes según el tipo de recurso. Esto haría que el código fuera más difícil de entender y mantener.

Si después se agregan más recursos, habría que continuar incorporando condiciones y relaciones, lo que podría aumentar todavía más el acoplamiento.

## 6. Comparación de alternativas

| Opción | Qué modifica | Qué reutiliza o duplica | Costo de agregar otro recurso | Riesgo sobre lo existente | Complejidad |
| --- | --- | --- | --- | --- | --- |
| **1. Módulo independiente** | Agrega tablas, archivos y rutas nuevas. | Reutiliza la autenticación y la estructura por capas, pero repite parte de la lógica de préstamos. | Alto: normalmente habría que construir otro módulo. | Bajo: los préstamos de libros pueden mantenerse sin modificaciones. | Baja a media. |
| **2. Sistema general de recursos** | Cambia el modelo de datos y una parte importante del módulo de préstamos. | Comparte la lógica común y reduce la duplicación. | Bajo a medio: se aprovecha la estructura general, aunque cada recurso puede tener reglas propias. | Alto: modifica el funcionamiento actual de los préstamos. | Alta al inicio. |
| **3. Ampliar el préstamo actual** | Modifica la tabla de préstamos, las consultas y las validaciones existentes. | Reutiliza el módulo actual, pero agrega condiciones para cada tipo de recurso. | Medio a alto: cada recurso añade más cambios y condiciones. | Medio a alto: interviene directamente en el código existente. | Media inicialmente, con posibilidad de aumentar. |

## 7. Conclusión del análisis

La revisión del código permitió identificar que el préstamo de libros depende de campos, consultas y reglas diseñadas para ese recurso. Esta dependencia facilita el manejo de libros, pero dificulta agregar laptops u otros elementos sin modificar varias partes del sistema.

Las tres alternativas permiten cumplir el nuevo requerimiento, aunque tienen costos diferentes. Un módulo independiente reduce los cambios en el sistema actual, pero puede duplicar código. Un sistema general facilita futuras ampliaciones, aunque necesita una modificación mayor. Ampliar el préstamo existente reutiliza componentes, pero aumenta las condiciones dentro de un mismo módulo.

Por lo tanto, la decisión no depende solamente de cuál opción permita agregar laptops, sino también del riesgo de afectar los préstamos actuales, la facilidad de mantenimiento y la posibilidad de agregar nuevos recursos en el futuro. Estos criterios se tendrán en cuenta para justificar la decisión arquitectónica en el ADR.
