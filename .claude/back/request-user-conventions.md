# Convenciones de `request.user`

Patrón extraído de `JwtAuthGuard` y `RolesGuard` (`src/modules/auth/guards/`
en `pcbox-api`). Aplica a cualquier guard o controller nuevo que necesite
leer al usuario autenticado.

## 1. Un solo escritor: `JwtAuthGuard`

`request.user` **solo** se asigna en `JwtAuthGuard`, después de verificar
el JWT con éxito:

```ts
request.user = payload; // JwtAuthGuard.canActivate
```

Ningún otro guard ni controller escribe `request.user` — todos los demás
son lectores. Esto mantiene un único punto donde "usuario autenticado"
puede volverse inválido o desactualizado.

El tipado de `request.user` no vive en este guard: viene de la
ampliación global de `Express.Request` en
`value-objects/express-request.d.ts` (ver `value-objects-conventions.md`).
Gracias a eso, la asignación es directa, sin castear el `request`.

## 2. El orden de `@UseGuards` importa

Nest ejecuta los guards de un `@UseGuards(...)` en el orden que están
declarados. `RolesGuard` asume que `request.user` ya existe cuando corre
— por eso `JwtAuthGuard` siempre va **primero**:

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
```

(ver `pcbox.controller.ts`). Invertir el orden rompe `RolesGuard` en
runtime, no en compilación — `request.user` es `optional` en el tipo
(`user?: AuthenticatedUser`) justo para que TypeScript no lo asuma
presente y obligue a chequearlo (siguiente punto).

## 3. Todo lector chequea presencia antes de usarlo

Como el tipo es `user?: AuthenticatedUser`, cualquier guard/handler que
lea `request.user` valida antes de usarlo y falla con un mensaje que
identifique la causa real (guard mal ordenado o ausente), no un error
genérico:

```ts
if (!request.user) {
  throw new ForbiddenException(
    'RolesGuard ran without an authenticated user - JwtAuthGuard must run first',
  );
}
```

(ver `roles.guard.ts`). El mensaje es intencionalmente específico — un
`403` pelado ahí sería indistinguible de un problema real de permisos.

## 4. Nunca leer `request.user` en un endpoint sin `JwtAuthGuard`

Un controller/handler que no tiene `JwtAuthGuard` en su cadena de guards
no debe asumir `request.user` — va a ser siempre `undefined`. Si un
endpoint nuevo necesita el usuario autenticado, agregar `JwtAuthGuard`
(solo o junto con `RolesGuard`, según si también necesita roles) es un
requisito, no un detalle de implementación.
