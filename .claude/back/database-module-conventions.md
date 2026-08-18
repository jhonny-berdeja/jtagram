# Convenciones de `common/database/database.module.ts`

Todo backend del ecosistema resuelve la conexión a Postgres y el
registro de entidades/repositories de la misma forma, en un único
`DatabaseModule` global. Aplica también a cualquier backend nuevo que
se agregue más adelante — copiar este patrón tal cual, no reinventarlo
por proyecto.

## Forma del archivo

```ts
import { Global, Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
// ... un import de entity + uno de repository por cada agregado.

@Global()
@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get<string>('DATABASE_HOST'),
        port: parseInt(
          configService.get<string>('DATABASE_PORT') ?? '5432',
          10,
        ),
        username: configService.get<string>('POSTGRES_USER'),
        password: configService.get<string>('POSTGRES_PASSWORD'),
        database: configService.get<string>('DATABASE_NAME'),
        entities: [/* ...todas las entities de la app */],
        synchronize: false,
      }),
    }),
    TypeOrmModule.forFeature([/* ...las mismas entities */]),
  ],
  providers: [/* ...un repository por entity/agregado */],
  exports: [/* ...los mismos repositories */, TypeOrmModule],
})
export class DatabaseModule {}
```

- `@Global()`: cualquier módulo de la app necesita sus repositories de
  forma implícita — nadie importa `DatabaseModule` a mano, se carga una
  sola vez en `AppModule`.
- `EnvModule` **no** se importa acá pese a que `inject: [ConfigService]`
  lo necesita: `EnvModule` también es `@Global()` (ver
  `env-config-conventions.md`), así que una vez cargado en `AppModule`,
  `ConfigService` ya es inyectable en cualquier lado sin import
  explícito — mismo motivo por el que ningún módulo de funcionalidad
  importa `DatabaseModule` para conseguir sus repositories.
- `synchronize: false` siempre: el schema lo gestiona una migración a
  mano (ver la documentación de deploy de cada base), nunca TypeORM
  auto-sincronizando contra una tabla en producción.
- `port` usa `parseInt(..., 10)` con `'5432'` como fallback textual
  antes de convertir — no como número — porque `configService.get()`
  siempre devuelve `string | undefined`.
- Las mismas entities aparecen tres veces (`entities` del
  `forRootAsync`, `TypeOrmModule.forFeature()`, y sus repositories en
  `providers`/`exports`) — no es redundancia accidental, son tres
  registros con propósitos distintos: la primera lista le dice a
  TypeORM qué tablas existen para la conexión real; `forFeature()`
  habilita `@InjectRepository()` para esas entities dentro de este
  módulo; `providers`/`exports` son los repositories propios del
  proyecto (no el `Repository<T>` crudo de TypeORM) que el resto de la
  app realmente inyecta.

## Qué NO va en el comentario del archivo

Todo lo de arriba es boilerplate — no hace falta repetirlo en cada
`database.module.ts` nuevo, esta es la referencia. Lo que sí amerita un
comentario propio en el archivo de una app puntual:

- **Por qué una entity específica está ahí**, cuando no es obvio por su
  nombre — p. ej. una tabla que existe para separar "acceso sin rol" de
  "acceso con rol" en vez de una columna nullable.
- **Historia de una entity que se fue**, cuando una migración retiró
  tablas que antes vivían acá — para que alguien leyendo el código en
  medio de esa migración entienda el porqué sin ir a buscar el commit.

El resto de las reglas para mantener las entities sincronizadas con el
schema real de cada base ya están en `project-structure-conventions.md`
— no se duplican acá.
