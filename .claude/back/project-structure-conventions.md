# ticket-hub-api — convenciones de estructura

## Organización de carpetas

- `src/common/` — infraestructura transversal, nunca específica de una
  funcionalidad:
  - `config/` — validación de variables de entorno (`env.validation.ts`)
    + el `EnvModule` global.
  - `database/<entidad>/` — una carpeta por entidad raíz, con su entity de
    TypeORM **y** su repository juntos. No separar por tipo de artefacto
    (`entities/`, `repositories/`) — todo lo referido a una entidad vive
    en una sola carpeta.
  - `dto/` — solo formas de respuesta genéricas y reutilizables (p. ej.
    `ResponseBody<T>`). Los DTOs específicos de una funcionalidad NO van
    acá — ver más abajo.
- `src/instrument/<concern>/` — infraestructura de *observability*, un
  bucket a nivel raíz aparte de `common/`: cada `<concern>` (`logger/` hoy,
  potencialmente `tracing/`, `metrics/` más adelante) es consumido de
  forma implícita por *toda* la app, no infraestructura de dominio o
  transversal que un módulo puntual podría o no necesitar. Mismo criterio
  interno que `common/config/` — separar la construcción pura de opciones
  (`logger.config.ts`) del wiring del módulo (`logger.module.ts`).
- `src/modules/<funcionalidad>/` — una carpeta por funcionalidad de
  negocio (`auth`, `users`, `tickets`, ...). Cada una con controller
  tiene `<funcionalidad>.module.ts`, `<funcionalidad>.controller.ts`,
  `<funcionalidad>.service.ts`
  - `dto/` — DTOs específicos del contrato HTTP de esa funcionalidad:
    tanto de request (`CreateUserDto`) como de response (`ResponseLogin`).
  - mappers (`<entidad>.mapper.ts`) y value objects internos que nunca
    cruzan el límite HTTP tal cual (`payload-jwt.ts`) van sueltos en la
    raíz del módulo, no dentro de `dto/`.
  - Un módulo que solo habla con un servicio externo, sin exponer nada
    por HTTP (`pcbox-api/` hoy), no tiene controller — ver "Patrón
    connector vs. service" más abajo.
- `test/` espeja a `src/`:
  - `test/common/` — infraestructura de test compartida (módulo de DB en
    memoria, helper de bootstrap de la app). Nada específico de una
    funcionalidad va acá.
  - `test/modules/<funcionalidad>/` — los e2e specs de esa funcionalidad.

## Reglas de dependencia entre módulos

- `@Global()` se reserva para los módulos que toda la app necesita de
  forma implícita (`EnvModule`, `DatabaseModule`). Siempre documentar, en
  un comentario sobre el módulo, *por qué* es global y *por qué* sus
  consumidores no lo importan explícitamente.
- Los módulos de funcionalidad pueden depender entre sí en una sola
  dirección, nunca de forma circular. Documentar la dirección y el motivo
  en un comentario de ambos lados (ver `auth.module.ts` /
  `users.module.ts`).

## Patrón builder

Toda clase con 2 o más campos obligatorios que represente un valor
construido —entities, DTOs transversales, value objects internos— lleva
un builder: un `.builder()` estático que devuelve una clase builder con
campos privados, setters fluidos `withX()`, y un `build()` que lanza error
si falta algún campo obligatorio. Forma de referencia: `UserEntity`,
`ResponseBody`, `PayloadJwt`.

- Si el valor construido termina en manos de algo que verifica en runtime
  que sea un objeto plano (p. ej. `jsonwebtoken.sign()`, que rechaza
  instancias de clase vía `lodash.isPlainObject`), `build()` debe devolver
  un objeto literal plano, no `new NombreClase()` — ver
  `PayloadJwtBuilder.build()` para el porqué y el cómo.

## Patrón de mappers

Cada entidad/agregado que se expone por HTTP tiene un mapper dedicado
(`<Entidad>Mapper`), ubicado en la raíz del módulo (no en `dto/`), con
métodos **únicamente `static`** — funciones puras, sin I/O ni estado.

- Un método por dirección de transformación, nombrado `to<Destino>`:
  - `toEntity(dto, ...valoresYaDerivados)` — DTO de entrada (+ algún
    valor calculado antes de llamar al mapper, como un password ya
    hasheado) → entidad persistible, construida vía el `.builder()` de
    la entidad.
  - `to<Operación>Fields(dto)` (p. ej. `toUpdateFields`) — DTO → un
    `Pick<Entidad, ...>` acotado a *solo* los campos que esa operación
    puntual tiene permitido tocar.
  - `toResponse(entity, ...datosRelacionados)` — entidad persistida (+
    datos relacionados, si los hay) → forma pública de respuesta. Acá es
    donde se descartan los campos que nunca deben salir (p. ej.
    `password`).
- Cada operación de escritura (create, update, ...) tiene su **propio**
  método de mapeo — nunca uno genérico reutilizado entre operaciones
  distintas. Aunque compartan campos, cada método documenta con su
  nombre y su tipo de retorno exactamente qué campos esa operación tiene
  permitido tocar (p. ej. un update de perfil no debe poder tocar
  `password` ni `roles` porque esos van por flujos propios).
- El service nunca arma objetos de entidad o de respuesta a mano,
  inline. El flujo siempre es: buscar/validar → derivar (hash, etc.) →
  mapear a entidad/campos con el mapper → llamar al repository → mapear
  el resultado a respuesta con el mapper. El mapper es la única costura
  entre "entidad" y "DTO/respuesta". Referencia: `UserMapper`.

## Handlers en el service general vs. service dedicado

El service general de un módulo (`<Módulo>Service`) reúne los métodos
handler de sus casos de uso (uno por endpoint). Cuándo un handler se
queda ahí y cuándo se muda a su propio archivo depende de si necesita
apoyarse en helpers privados propios:

- **Handler autocontenido** (solo orquesta llamadas a repository/mapper/
  otros services inyectados, sin métodos privados propios) → se queda en
  el service general del módulo. Ejemplos: `UsersService.create`,
  `UsersService.findAll`, `AuthService.login`.
- **Handler que llama a helpers privados definidos en el mismo archivo
  solo para él** → el handler completo (método público + todos sus
  helpers privados) se muda a un archivo de service dedicado y exclusivo
  para esa operación. Convención de nombres: `<operación>-<entidad>.
  service.ts` con clase `<Operación><Entidad>Service` (p. ej.
  `update-user.service.ts` → `UpdateUserService`), registrado como
  provider en el módulo y usado directamente por el controller para ese
  endpoint puntual — el service general no delega en él ni lo envuelve.
  Referencia: `UpdateUserService` (extraído de `UsersService.update`
  porque dependía de `findExistingUserOrThrow` y
  `assertEmailNotTakenByAnotherUser`).

El objetivo es que revisar un handler simple no obligue a leer el
archivo general completo, y que un handler con lógica auxiliar propia
quede aislado en un archivo chico y fácil de revisar solo.

## Patrón "connector vs. service" (para módulos que solo hablan con algo externo)

Un módulo cuyo único trabajo es hablarle a un servicio externo, sin
exponer nada por HTTP propio (`pcbox-api/` hoy — le pega a la API real de
`pcbox-api` el otro repo, nombre igual pero cosas distintas), repite el
mismo split que ya usa `pcbox-api` (el repo) para sus propios módulos
`ansible/`/`ticket-hub-api/`: un `<Módulo>Connector` que es pura mecánica
de I/O (arma la URL, el header del secreto, el timeout, no decide nada
de negocio) y un `<Módulo>Service` que es la API pública del módulo —
decide *qué* mandar y *qué significa* la respuesta, nunca hace el I/O él
mismo.

- `PcboxApiConnector` — arma la URL completa, adjunta `x-admin-api-key`,
  aplica el timeout, devuelve el `Response` crudo o rechaza.
  `PcboxApiService` decide qué campos mandar (resolviendo
  `creator`/`assignee` a nombres vía `UsersRepository`, sin ningún
  round-trip HTTP — esta app tiene acceso directo a esa tabla) y qué
  hacer con la respuesta.

El connector es siempre provider del módulo pero **nunca exportado** —
solo el service lo es (ver `PcboxApiModule`). Nada fuera del módulo toca
el connector directo.

## Variables de entorno

`src/common/config/env.validation.ts` es la única fuente de verdad para
las variables de entorno requeridas (todas obligatorias, validadas con
`class-validator`). Cualquier variable que se agregue, quite o renombre
ahí debe reflejarse tanto en `.env.example` como en la tabla del
`README.md` — las tres nunca deben desincronizarse.
