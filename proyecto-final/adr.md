# ADR-001. Incorporación del préstamo de laptops

**Estado:** Aceptado  
**Fecha:** 23 de noviembre de 2026

## Contexto

La API de la biblioteca permite prestar libros, pero el préstamo actual depende directamente de la tabla `books`. La tabla `loans` exige un `book_id`, y el servicio de préstamos revisa las copias disponibles antes de registrar cada operación. Por eso, no basta con agregar laptops al inventario: también es necesario decidir cómo se van a manejar sus préstamos.

El nuevo módulo debe permitir registrar laptops, consultar su disponibilidad, prestarlas, devolverlas y ver los préstamos del usuario. Al mismo tiempo, deben seguir funcionando la autenticación y los préstamos de libros.

En el análisis inicial se consideraron tres alternativas:

1. **Crear un módulo independiente para laptops:** agregar sus propias tablas y capas sin modificar el módulo de préstamos de libros.
2. **Crear un sistema general de recursos:** adaptar los préstamos para que puedan manejar libros, laptops y otros recursos con una estructura común.
3. **Ampliar el préstamo actual:** modificar las tablas y el servicio existente para distinguir entre libros y laptops mediante campos y condiciones.

Para tomar la decisión se tuvieron en cuenta cuatro aspectos: el riesgo de afectar funciones existentes, la facilidad de implementación, el mantenimiento del código y la posibilidad de agregar nuevos recursos más adelante.

## Decisión

Se decidió **crear un módulo independiente para el préstamo de laptops**, siguiendo la arquitectura por capas que ya utiliza la API: *handler*, *service* y *repository*.

Se agregarán tablas nuevas para el inventario de laptops y sus préstamos, relacionadas con los usuarios existentes. También se crearán modelos, servicios, repositorios, handlers y rutas propias. Se reutilizará la autenticación con JWT y la validación de roles, sin modificar las tablas `books` y `loans` ni cambiar las reglas de préstamo de libros.

Esta opción se eligió principalmente porque permite incorporar la nueva función con menos riesgo de afectar lo que ya funciona. También facilita manejar reglas propias de las laptops sin añadir condiciones al servicio de libros.

Aunque un sistema general de recursos podría ser más útil si la biblioteca agregara muchos tipos de préstamos, implementarlo ahora exigiría cambiar varias partes del proyecto y realizar más pruebas sobre las funciones existentes. Por otro lado, ampliar el módulo actual reduciría algunos archivos nuevos, pero mezclaría reglas de libros y laptops dentro del mismo servicio. Por estas razones, las dos alternativas se descartaron para este proyecto.

### Reglas para el préstamo de laptops

- **Inventario individual:** cada laptop tendrá un registro propio y un número de serie único. Esto permitirá saber exactamente qué equipo está prestado, a diferencia de los libros, que se controlan por cantidad de copias.
- **Disponibilidad:** una laptop solo podrá tener un préstamo activo a la vez. Al prestarla quedará como no disponible, y al devolverla volverá a estar disponible.
- **Duración:** se establece un plazo de **7 días** para devolverla. Es una regla propuesta para diferenciar el préstamo de equipos del de libros, cuyo plazo actual es de 14 días.
- **Permisos:** solo los administradores podrán registrar laptops. Los usuarios autenticados podrán consultar el inventario, solicitar préstamos, devolverlos y consultar sus propios préstamos, siguiendo los controles de acceso que usa actualmente la API.
- **Registro seguro:** el préstamo y el cambio de disponibilidad se realizarán dentro de una misma transacción en la base de datos. Así se evita que una operación quede guardada y la otra no. También se comprobará que el equipo siga disponible al momento de prestarlo, para impedir préstamos simultáneos de la misma laptop.

## Consecuencias

### Ventajas

- **Menor riesgo sobre el sistema actual:** no es necesario cambiar las tablas ni la lógica de los préstamos de libros.
- **Código más organizado:** las reglas de laptops estarán en archivos propios y no mezcladas con las de libros.
- **Aprovechamiento de lo existente:** se mantienen la autenticación, los roles, la conexión a MySQL y la estructura por capas.
- **Control específico del inventario:** cada equipo podrá identificarse por su serial y su estado de disponibilidad.

### Desventajas

- **Parte del código se repetirá:** prestar, devolver y listar préstamos son operaciones parecidas a las que ya existen para libros.
- **Agregar otros recursos llevará más trabajo:** si después se prestan proyectores o tablets, probablemente habrá que crear módulos adicionales o reconsiderar una estructura compartida.
- **Habrá más archivos y tablas:** el proyecto será un poco más grande y cada módulo necesitará mantenimiento y pruebas.

## Efecto sobre el sistema y posibles cambios futuros

La base de datos conservará las tablas existentes y sumará las de laptops y préstamos de laptops mediante una nueva migración. Dentro de la API, el nuevo módulo tendrá sus propias capas, pero compartirá usuarios, autenticación y conexión a MySQL con el resto del sistema.

Si en el futuro la biblioteca empieza a prestar muchos tipos de recursos, sería conveniente volver a evaluar un modelo de préstamos más general. Por ahora, la prioridad es agregar laptops de manera clara y sin poner en riesgo el funcionamiento de los libros. Esta decisión no elimina la posibilidad de cambiar la arquitectura más adelante, pero evita hacer una modificación mayor antes de que realmente sea necesaria.
