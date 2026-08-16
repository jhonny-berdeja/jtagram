# Convenciones de Exception Filters (manejo de errores global)

Patrón extraído de los tres filtros existentes en `src/common/filters/`
(`DatabaseExceptionFilter`, `HttpExceptionFilter`, `UnknownExceptionFilter`).
Aplica a cualquier filtro global nuevo que se agregue a este set.

## 1. Nunca `err: exception` crudo

Pino tiene un serializer especial para la clave `err`: cuando la ve, hace
`for (const key in err)` y copia **cualquier propiedad enumerable** del
objeto al log — no solo `message`/`stack`. Las clases de excepción de este
proyecto (y de sus dependencias) guardan datos que no deberían salir así
nomás:

- `QueryFailedError` (TypeORM) guarda `.query`, `.parameters` (los valores
  bindeados de la query — pueden ser un hash de password, un email) y
  además copia sobre sí misma **todas** las propiedades del error del
  driver (en Postgres real, eso incluye `.detail`, que en un unique
  constraint trae el valor literal).
- `HttpException` (Nest) guarda `.response`/`.status`/`.options`, y
  `.options.cause` puede envolver cualquier error interno arbitrario.

Ninguna regla de `redact` alcanza para taparlo de forma confiable — el
nombre exacto del campo depende de qué tiró el error, y varía. La forma
correcta es no entregarle a pino el objeto crudo en absoluto: armar el
`err` a mano, con solo lo que se sabe que es seguro:

```ts
err: {
  message: exception.message,
  stack: exception.stack,
},
```

Esto hace el leak estructuralmente imposible, en vez de depender de
acordarse de redactar cada campo nuevo que un driver/librería decida
agregar.

## 2. `errorType` va como campo hermano, nunca adentro de `err`

Pino siempre pisa `err.type` con `err.constructor.name` — como el `err`
que armamos a mano es un objeto plano (`{message, stack}`), ese valor
siempre termina siendo `"Object"`, inútil para filtrar en Loki/Grafana. La
subclase real (`QueryFailedError`, `ConflictException`, `TypeError`, ...)
va en un campo separado que pino no toca:

```ts
errorType: exception.constructor.name,
```

## 3. `msg` va adentro del objeto, no como segundo argumento

`nestjs-pino`'s `Logger` sigue la convención de `LoggerService` de Nest —
`error(message, ...optionalParams)`, donde el **último** argumento extra
se interpreta como `context`, no como el mensaje del log. Pasar un string
como segundo argumento (`this.logger.error({...}, 'Database error')`)
termina guardado en un campo `context`, y pino cae al fallback de usar
`err.message` como `msg`. Para que el mensaje que se quiere sea el `msg`
real, tiene que ir como clave dentro del mismo objeto, en un único
argumento:

```ts
this.logger.error({
  err: { message: ..., stack: ... },
  errorType: ...,
  msg: 'Database error', // acá, no como segundo argumento
});
```

## 4. Separar qué le llega al cliente de qué llega a Loki

Dos casos, según si el mensaje de la excepción ya está pensado para el
usuario o no:

- **Detalle interno (nunca sale)**: `DatabaseExceptionFilter`,
  `UnknownExceptionFilter`. El cliente recibe siempre
  `GENERIC_ERROR_MESSAGE` (constante compartida en
  `generic-error-message.ts`, para que el texto no pueda desincronizarse
  entre filtros) + `500`. Loki recibe el detalle completo.
- **Mensaje ya pensado para el usuario (sale intacto)**:
  `HttpExceptionFilter`. `ConflictException('Email already in use')` ya
  es un mensaje deliberado — no hay nada que esconder. El filtro solo
  agrega logging, la respuesta es exactamente
  `response.status(status).json(exception.getResponse())`, lo mismo que
  mandaría Nest por default sin el filtro.

## 5. Nivel de log según severidad, no fijo en `error`

Un filtro que loguea todo como `error` hace ruido inútil: un `409` por
email duplicado no es un bug, es tráfico esperado. `HttpExceptionFilter`
resuelve esto con el status: `>= 500` → `error` (algo se rompió de
verdad), `< 500` → `warn` (esperado, no debería disparar una alerta).
`DatabaseExceptionFilter`/`UnknownExceptionFilter` siempre son `error`
porque, por definición, lo que atrapan nunca fue anticipado.

## 6. Orden de registro en `providers` — el gotcha crítico

Un filtro con `@Catch()` sin argumento (como `UnknownExceptionFilter`)
matchea *cualquier* excepción. Nest resuelve cuál filtro usar con
`Array.prototype.find` sobre la lista de filtros globales — el primero
que matchea, gana. Pero esa lista **no** respeta el orden de declaración
en `providers`: `RouterExceptionFilters` la invierte
(`filters.reverse()`, en
`@nestjs/core/router/router-exception-filters.js`) antes de pasarla a
`find`. Resultado: lo que se declara **primero** en `providers` es lo
**último** que se prueba en runtime.

```ts
providers: [
  // Va PRIMERO en la declaración para terminar ÚLTIMO en el matching.
  { provide: APP_FILTER, useClass: UnknownExceptionFilter },
  // Entre estos dos el orden no importa — son clases de excepción
  // disjuntas, cada uno matchea solo lo suyo.
  { provide: APP_FILTER, useClass: DatabaseExceptionFilter },
  { provide: APP_FILTER, useClass: HttpExceptionFilter },
],
```

Esto no es intuitivo y es fácil de romper en silencio (un filtro
`@Catch()` mal ubicado se come a todos los demás sin tirar ningún error).
Si se agrega o reordena algo acá, no alcanza con leer el código fuente de
Nest para confirmarlo — armar una app de Nest real mínima, tirar cada tipo
de excepción, y confirmar en qué filtro cae cada una (así se encontró y
confirmó este comportamiento la primera vez).

## 7. Ubicación y tests

- Filtro: `src/common/filters/<nombre>.filter.ts`.
- Test: `src/common/filters/test/<nombre>.filter.spec.ts` (no colocado
  al lado del filtro — a diferencia del resto del repo, donde los
  `.spec.ts` sí van colocados junto a su fuente, ver
  `src/instrument/logger/logger.config.spec.ts`).
- Cada test arma su propio `buildHost()` que mockea
  `ArgumentsHost.switchToHttp().getResponse()` (y `.getRequest()` cuando
  aplica), sin levantar una app de Nest real — más rápido, y suficiente
  porque `catch()` es una función pura respecto a sus argumentos.
