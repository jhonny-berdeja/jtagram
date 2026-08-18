# Convenciones de configuración de variables de entorno

Patrón extraído de `src/common/config/` en los tres backends del
ecosistema (`auth-api`, `ticket-hub-api`, `pcbox-api`) — `env.module.ts`
y `env.validation.ts` son estructuralmente idénticos en los tres,
variable por variable propia de cada app aparte. Aplica a cualquier
backend nuevo que se agregue.

## 1. `EnvModule` — wrapper global de `ConfigModule`

Siempre el mismo archivo, sin variación entre apps:

```ts
import { Global, Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { validate } from './env.validation';

@Global()
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      ignoreEnvFile: true,
      validate,
    }),
  ],
  exports: [ConfigModule],
})
export class EnvModule {}
```

- `@Global()` porque `ConfigService` lo necesita implícitamente cualquier
  módulo de la app (incluido `DatabaseModule`) — nadie importa
  `EnvModule` a mano, solo se carga una vez en `AppModule`.
- `ignoreEnvFile: true` es deliberado: todas las apps corren
  **cluster-only**, cada valor llega como variable de entorno inyectada
  por el Deployment/Secret de Kubernetes, nunca por un archivo `.env`.

Ningún backend nuevo necesita comentarios propios en este archivo — es
boilerplate puro, la explicación vive acá.

## 2. `env.validation.ts` — única fuente de verdad, todo obligatorio

Misma forma en los tres backends:

```ts
import { plainToInstance } from 'class-transformer';
import {
  IsIn,
  IsNotEmpty,
  IsNumberString,
  IsString,
  validateSync,
} from 'class-validator';

const PINO_LOG_LEVELS = [
  'trace', 'debug', 'info', 'warn', 'error', 'fatal',
] as const;

export class EnvironmentVariables {
  // ... una propiedad por variable, siempre @IsString()/@IsNotEmpty()
  // salvo los dos casos especiales de abajo.
}

export function validate(
  config: Record<string, unknown>,
): EnvironmentVariables {
  const validatedConfig = plainToInstance(EnvironmentVariables, config);

  const errors = validateSync(validatedConfig, {
    skipMissingProperties: false,
  });

  if (errors.length > 0) {
    const invalid = errors.map((error) => error.property).join(', ');
    throw new Error(`Missing required environment variable(s): ${invalid}`);
  }

  return validatedConfig;
}
```

- **Todas las variables son obligatorias** — no hay defaults ni
  variables opcionales. Si algo falta, `validate()` tira al arrancar
  (fail-fast), en vez de que la app arranque con un valor tipo
  `localhost` silenciosamente mal.
- **Dos únicas excepciones** al `@IsString()/@IsNotEmpty()` genérico:
  - `DATABASE_PORT`/`PORT` van con `@IsNumberString()` — se les hace
    `parseInt()` o se pasan directo a `app.listen()` más abajo en el
    bootstrap, así que un valor no numérico tiene que fallar acá, no
    ahí.
  - `LOG_LEVEL` va con `@IsIn(PINO_LOG_LEVELS)` contra el array de
    niveles válidos de pino, en orden de severidad ascendente
    (`trace/debug/info/warn/error/fatal`).
- `validate()` en sí (imports, cuerpo de la función, el mensaje de
  error) es idéntico en los tres backends — no hace falta comentarlo de
  nuevo en cada repo nuevo, esta es la referencia.

## 3. Comentarios solo donde el nombre de la variable no basta

Una variable como `POSTGRES_USER`, `PORT` o `LOG_LEVEL` no necesita
comentario — el nombre y el tipo ya dicen todo. Documentar en el lugar
de la propiedad **solo** cuando hace falta explicar algo que no se
deduce del nombre:

- **De dónde sale el valor** si no es obvio (p. ej. `AUTH_API_URL` →
  DNS in-cluster de otro namespace; `JWT_PRIVATE_KEY`/`JWT_PUBLIC_KEY` →
  ver el doc de generación de claves).
- **Por qué esta app y no otra la necesita**, cuando el nombre podría
  sugerir que es compartida (p. ej. `PCBOX_API_APPLICATION_NAME`: el
  nombre de aplicación contra el que este backend se loguea en
  `auth-api`, no el nombre de la app en sí).
- **Qué reemplazó**, cuando la variable vino a sustituir un mecanismo
  viejo (p. ej. `AUTH_API_URL` reemplazando un `ADMIN_API_KEY`
  compartido) — así alguien que lee el código en el medio de una
  migración entiende el porqué sin tener que ir a buscar el commit.

El resto de las reglas para mantener `env.validation.ts` sincronizado
con el Secret/README de cada app ya están en
`project-structure-conventions.md`, sección "Variables de entorno" — no
se duplican acá.
