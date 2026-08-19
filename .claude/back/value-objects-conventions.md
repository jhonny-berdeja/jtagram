# Convenciones de `value-objects/`

Concreta, con un ejemplo real, la regla que ya menciona
`project-structure-conventions.md`: los value objects internos que nunca
cruzan el límite HTTP tal cual (ejemplo de referencia ahí: `payload-jwt.ts`)
van sueltos en la raíz del módulo, no dentro de `dto/`. Cuando un módulo
acumula más de uno, se agrupan en una carpeta `value-objects/` dentro de
ese módulo — no una carpeta compartida a nivel `common/`, porque estos
tipos son específicos de la funcionalidad que los define.

Caso real: `src/modules/auth/value-objects/` (`pcbox-api`), con
`authenticated-user.ts` y `role.enum.ts`.

## Qué va acá vs. qué va en `dto/`

El criterio no es "interface vs. class" ni "tiene decoradores de
`class-validator`" — es si el objeto se serializa tal cual como body de un
request/response HTTP:

- **`dto/`**: contrato HTTP explícito de la funcionalidad. Si mañana se
  loguea o se imprime tal cual, es literalmente lo que el cliente mandó o
  lo que el cliente recibe.
- **`value-objects/`**: forma interna que un flujo usa entre sí, pero que
  nunca es la respuesta ni el body de un endpoint. `AuthenticatedUser` es
  el payload ya decodificado de un JWT (`JwtAuthGuard` lo cuelga en
  `request.user`, ver `jwt-auth.guard.ts`) — ningún endpoint lo devuelve
  como está. `Role` es un enum de autorización que se usa en decoradores y
  guards (`@Roles(Role.ADMIN)`), no un campo que un cliente mande o
  reciba en el body.

## Ubicación y naming

- `src/modules/<funcionalidad>/value-objects/<nombre>.ts`.
- Sin sufijo obligatorio en el archivo (a diferencia de `dto/`, que
  siempre termina en `.dto.ts`) — el nombre describe el concepto tal cual:
  `authenticated-user.ts`, `role.enum.ts`.

## Declaraciones de ambiente global (`.d.ts`)

Un caso borde: `express-request.d.ts` (ampliación global de
`Express.Request` para tipar `request.user: AuthenticatedUser` sin castear
en cada guard) también vive en `value-objects/`, aunque estrictamente no
es un value object en sí — es una declaración ambiente que *referencia*
uno. Se tolera ahí porque la carpeta es chica y el archivo solo tiene
sentido junto a `AuthenticatedUser`. Si `value-objects/` crece y empieza a
mezclar tipos de negocio con declaraciones de ambiente, separar en algo
como `value-objects/` vs. `types/` (o un archivo `<módulo>.d.ts` suelto en
la raíz del módulo) deja de ser prematuro.

```ts
// express-request.d.ts
import { AuthenticatedUser } from './authenticated-user';

declare global {
  namespace Express {
    interface Request {
      user?: AuthenticatedUser;
    }
  }
}
```

Con esto, cualquier guard o controller que lea `request.user` lo tiene
tipado sin `as Request & { user: AuthenticatedUser }` repetido en cada
punto de uso — ver `roles.guard.ts` y `jwt-auth.guard.ts`.
