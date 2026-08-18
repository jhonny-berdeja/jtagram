# Convenciones de `src/instrument/logger/`

Patrón extraído de los archivos de logger de las tres apps backend
(`logger.config.ts` y `logger.module.ts` en las tres; `logger.config.spec.ts`
en pcbox-api y ticket-hub-api) en auth-api, pcbox-api y ticket-hub-api. Aplica
a cualquier backend nuevo que se agregue al ecosistema — copiar la forma tal
cual, no reinventarla por proyecto.

## 1. `buildLoggerOptions(level)` — forma del archivo

Los tres `logger.config.ts` son el mismo archivo, con una sola diferencia de
un campo (ver sección 3):

```ts
import { randomUUID } from 'node:crypto';
import { Options } from 'pino-http';

export function buildLoggerOptions(level: string): { pinoHttp: Options } {
  return {
    pinoHttp: {
      level,
      genReqId: () => randomUUID(),
      redact: {
        paths: [
          'req.headers.authorization',
          'req.headers.cookie',
          'res.headers["set-cookie"]',
          'req.body.password',
          'req.body.access_token',
          'err.parameters',
        ],
        censor: '[REDACTED]',
      },
    },
  };
}
```

`level` llega ya validado desde `LOG_LEVEL` (`@IsIn(PINO_LOG_LEVELS)` en
`env.validation.ts`, ver `env-config-conventions.md` sección 2) — este
archivo no valida nada, solo arma el objeto de opciones de `pino-http`. La
función existe separada del módulo (sección 5) específicamente para poder
testearla sin levantar Nest — ver sección 7.

## 2. `genReqId` con `randomUUID()`, no el contador default de `pino-http`

Sin `genReqId`, `pino-http` genera `req.id` con un contador secuencial en
memoria del proceso (`1`, `2`, `3`, ...). En este ecosistema eso es
inútil para correlacionar logs en Loki: cada app corre como Pod en
Kubernetes, con múltiples réplicas posibles y reinicios frecuentes, así que
un `req.id` secuencial se reinicia en `1` en cada restart y colisiona entre
réplicas — dos requests completamente distintos, en pods distintos, pueden
compartir el mismo `req.id`. `randomUUID()` da unicidad global sin depender
de que el proceso sea el único generándolos:

```ts
genReqId: () => randomUUID(),
```

Donde hay spec (pcbox-api, ticket-hub-api), esto está testeado explícitamente
contra el contrato de pino-http (`genReqId(req, res)`, no `genReqId()`), no
solo contra el formato del UUID:

```ts
const first = genReqId(undefined as never, undefined as never);
const second = genReqId(undefined as never, undefined as never);

expect(first).toMatch(uuidV4);
expect(second).toMatch(uuidV4);
expect(first).not.toBe(second);
```

## 3. Redact list — mismo shape, un campo que varía por dominio

Los seis paths de `redact.paths` son estructuralmente los mismos en las tres
apps — headers de credenciales en tránsito (`authorization`, `cookie`,
`set-cookie` de la respuesta), el token de acceso, y `err.parameters` (ver
sección 4) — salvo un campo, que cambia según qué credencial recibe cada app
en el body de login:

- pcbox-api y ticket-hub-api: `req.body.password`.
- auth-api: `req.body.clienteSecret` — no es un typo ni una app
  desincronizada; `apps-users` en auth-api se autentica con
  `clienteId`/`clienteSecret` (ver `login.dto.ts`), no con `password`, así
  que el campo a redactar es literalmente distinto.

Si se agrega un backend nuevo, el campo a redactar es el que reciba su propio
endpoint de login — no copiar `password` a ciegas si el dominio de esa app
usa otro nombre.

## 4. `err.parameters` — segunda barrera para el mismo leak que ya bloquean los filtros

`exception-filters-conventions.md` (sección 1) documenta por qué
`DatabaseExceptionFilter` arma `err` a mano (`{ message, stack }`) en vez de
pasarle a pino el `QueryFailedError` crudo: ese error de TypeORM trae
`.parameters` con los valores bindeados de la query, que pueden incluir un
hash de password o un email. Los tres filtros de este ecosistema respetan
esa disciplina — ninguno pasa el error crudo.

El redact rule de `err.parameters` en `logger.config.ts` no es redundante con
eso: cubre el camino que los filtros **no** controlan. `pino-http` loguea un
`err` automáticamente por su cuenta en cualquier response que termine en
`>= 500`, con esta lógica (`pino-http/logger.js`):

```js
if (err || res.err || res.statusCode >= 500) {
  const error = err || res.err || new Error('failed with status code ' + res.statusCode)
  // ...se serializa bajo la key `err`
}
```

Hoy ninguna de las tres apps asigna `res.err`, así que ese camino automático
cae al `Error` genérico de fallback (sin `.parameters`) — pero el redact
rule sigue siendo necesario como red de seguridad para dos escenarios reales:
que en el futuro algo asigne `res.err = exception` con el error de TypeORM
crudo, o que un contributor nuevo escriba en algún servicio
`this.logger.error({ err: exception })` pasando el `QueryFailedError` tal
cual, sin pasar por el filtro. En cualquiera de los dos casos, `err.parameters`
en el `redact.paths` tapa el mismo leak que la sección 1 de
`exception-filters-conventions.md` ya bloquea del lado de los filtros — es
la misma protección, duplicada a propósito en la capa de config del logger
para no depender de que todo el código futuro recuerde la regla de "armar
`err` a mano".

## 5. `logger.module.ts` — wiring y dónde vive la explicación canónica

Los tres módulos son la misma estructura:

```ts
@Global()
@Module({
  imports: [
    PinoLoggerModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (configService: ConfigService) =>
        buildLoggerOptions(configService.get<string>('LOG_LEVEL')!),
    }),
  ],
  exports: [PinoLoggerModule],
})
export class LoggerModule {}
```

- `@Global()` por el mismo motivo que `EnvModule`/`DatabaseModule`: cualquier
  parte de la app necesita loguear implícitamente, así que nadie debería
  tener que importar `LoggerModule` a mano en un feature module.
- El `!` en `configService.get<string>('LOG_LEVEL')!` es seguro porque
  `LOG_LEVEL` es una variable obligatoria validada en el arranque (ver
  sección 1) — no puede llegar `undefined` a un proceso que ya bootstrapeó.

Lo que **no** se repite igual en las tres apps es el comentario de
`LoggerModule`, y eso es deliberado, no inconsistencia: ticket-hub-api tiene
el texto canónico completo (por qué `nestjs-pino` sobre el logger interno de
Nest: JSON estructurado para Loki/Promtail, `app.useLogger()` cubriendo
también los logs del framework, y `req.id` automático vía `pino-http`).
pcbox-api copia ese mismo texto y le agrega un párrafo propio (por qué es
especialmente importante ahí: `AnsibleService` loguea el resultado completo
de cada playbook real contra `pcbox`, y ese es el audit trail de una acción
administrativa). auth-api, en cambio, no copia el texto — lo acorta y dice
explícitamente "Mirrors ticket-hub-api's ... logger.module.ts exactly — see
that file for the full library-choice rationale". Si se agrega un backend
nuevo, seguir ese mismo criterio: no repetir la explicación completa de cero,
referenciar ticket-hub-api como fuente y agregar solo el párrafo específico
de la app nueva si existe una razón puntual (como la de `AnsibleService`).

## 6. Bootstrap: `bufferLogs: true` + `app.useLogger()`

No vive en `src/instrument/logger/`, pero es la otra mitad obligatoria del
wiring, igual en los tres `main.ts`:

```ts
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(Logger));
```

`bufferLogs: true` retiene los logs propios de Nest (inicialización de
módulos, mapeo de rutas) hasta que `useLogger()` reemplace el logger default
por el de `nestjs-pino` — sin esto, esos logs de arranque salen con el
logger plano de Nest antes de que `nestjs-pino` esté listo, y terminan como
texto suelto en vez de JSON, rompiendo la garantía de "todo log de esta app
es JSON estructurado" que motiva el módulo entero (sección 5).

## 7. Tests

- Ubicación: `src/instrument/logger/logger.config.spec.ts`, colocado junto a
  su fuente (igual que el resto del repo — a diferencia de los filtros, ver
  `exception-filters-conventions.md` sección 7).
- pcbox-api y ticket-hub-api tienen la misma suite, palabra por palabra:
  nivel seteado desde `level`, los seis paths de redact presentes, y
  unicidad/formato de `genReqId`.
- `logger.module.ts` no tiene spec propio en ninguna de las tres apps: no
  hay nada que testear ahí sin levantar Nest completo (es wiring de
  `forRootAsync`, no lógica), así que toda la cobertura vive del lado de
  `buildLoggerOptions`, que sí es una función pura.
- auth-api no tiene `logger.config.spec.ts`. No es una omisión puntual del
  logger: auth-api no tiene **ningún** `.spec.ts` en todo `src/` todavía
  (proyecto sin suite de tests aún, no una decisión específica de este
  módulo). Al agregar tests a auth-api, copiar la spec de pcbox-api/
  ticket-hub-api tal cual, ajustando solo el campo de redact (`clienteSecret`
  en vez de `password`, ver sección 3).
