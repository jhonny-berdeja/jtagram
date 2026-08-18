# Convenciones de configuración de variables de entorno

Todo backend del ecosistema resuelve `src/common/config/` de la misma
forma: `env.module.ts` y la estructura de `env.validation.ts` son
siempre el mismo archivo, variable por variable propia de cada app
aparte. Aplica también a cualquier backend nuevo que se agregue más
adelante — copiar este patrón tal cual, no reinventarlo por proyecto.

## 1. `EnvModule` — wrapper global de `ConfigModule`

Siempre el mismo archivo, sin variación de una app a otra:

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

Misma forma en cualquier backend:

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
  error) se copia tal cual en cada backend nuevo — no hace falta
  comentarlo de nuevo en cada repo, esta es la referencia.

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
