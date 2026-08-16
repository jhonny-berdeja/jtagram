# Estructura de frontend (Next.js App Router)

Convención de organización de código para cualquier proyecto Next.js con
App Router. No es específica de este repo — aplica igual a cualquier
proyecto que use `app/` + componentes colocados.

## 🎯 Principio

`page.tsx` **solo** define la estructura general de la página y compone
componentes. No contiene lógica de negocio, fetching directo, ni estado
complejo. Toda esa lógica vive dentro de los componentes que `page.tsx`
invoca.

## 📁 Estructura de páginas

```
app/
└── users/
    ├── page.tsx        # Estructura de la página
    ├── layout.tsx       # Layout de la página (si aplica) — guards
    │                     # cross-cutting (auth, roles) van acá, no en
    │                     # cada page.tsx del subárbol
    └── components/      # Componentes propios de esta página
```

## 📁 Estructura de un componente

Cada componente tiene su propia carpeta, con archivos "hermanos" que
separan responsabilidades. Ejemplo con `CreateUserForm`:

```
components/
└── create-user-form/
    ├── CreateUserForm.tsx              # Implementación del componente
    ├── create-user-form.api.ts         # Requests a la API
    ├── create-user-form.dto.ts         # DTOs / builders
    ├── create-user-form.service.ts     # Lógica de negocio pura
    ├── create-user-form.provider.tsx   # CreateUserFormProvider
    ├── create-user-form.context.ts     # Definición del CreateUserFormContext
    ├── use-create-user-form.ts         # Hook useCreateUserForm
    └── components/                     # Componentes hijos (si los hay)
        └── ...
```

### Convención de nombres de archivo

| Archivo | Responsabilidad |
|---|---|
| `ComponentName.tsx` | Implementación visual/estructural del componente |
| `component-name.api.ts` | Llamadas a la API relacionadas a este componente |
| `component-name.dto.ts` | Clases de datos (patrón builder), modelos, mappers |
| `component-name.service.ts` | Lógica de negocio pura (sin React, sin llamadas a API): reglas derivadas sobre datos que el componente/hook ya tiene, testeables sin renderizar nada |
| `component-name.provider.tsx` | Implementación del `ComponentNameProvider` |
| `component-name.context.ts` | Creación del `ComponentNameContext` (`createContext`) |
| `use-component-name.ts` | Hook custom para consumir el contexto (`useComponentName`) |
| `components/` | Componentes hijos, cada uno con esta misma estructura |

**Regla de profundidad:** si un componente tiene componentes hijos, estos
van dentro de su propia carpeta `components/`, y cada hijo repite
exactamente esta misma estructura, anidada tantos niveles como haga falta.

**Archivo solo si hace falta:** una unidad (componente, o cualquier cosa
dentro de `common/`, ver más abajo) solo tiene los archivos hermanos que su
responsabilidad realmente requiere. Un componente que no llama API no
tiene `.api.ts`. Uno que no comparte estado no tiene `.provider.tsx` ni
`.context.ts`. No se crean por convención automática ni "por las dudas".

## 🚪 Rutas en vez de modales

**Antes de reachar para un modal + estado compartido entre "la lista" y
"el formulario", preguntate si esto no debería ser directamente una
ruta.** Un modal para crear/editar/ver un registro casi siempre termina
necesitando que dos componentes que se montan al mismo tiempo (la lista y
el formulario) se coordinen — y esa coordinación es exactamente lo que
generaba la necesidad de Context en este proyecto. Convertir esos flujos
en páginas separadas elimina el problema de raíz: si el formulario nunca
está montado al mismo tiempo que la lista, no hay nada que coordinar.

### Estructura típica de un CRUD con esta convención

```
app/home/users/
├── layout.tsx           # Guard (ej: solo admin), aplica a todo el subárbol
├── users.dto.ts          # Tipos de dominio compartidos por list/create/[id]
├── users.api.ts           # Requests compartidos por list/create/[id]
├── page.tsx                # Landing: solo dos <Link> a list/ y create/
├── list/
│   ├── page.tsx              # Fetchea la lista, self-contained
│   └── components/
│       └── users-table/       # Recibe datos por props (users, isLoading, error)
├── create/
│   ├── page.tsx                # Compone el form dentro de un card normal,
│   │                             # NO un overlay "fixed inset-0"
│   └── components/
│       └── create-user-form/    # Sin props onClose/onCreated — usa
│                                  # useRouter() y navega a list/ al
│                                  # terminar (éxito o "Cancelar")
└── [id]/
    └── edit/
        ├── page.tsx                # Ver "patrón sin GET-by-id" abajo
        └── components/
            └── edit-user-form/      # Recibe `user` por prop (su padre
                                       # directo ya lo resolvió), sin
                                       # onClose/onSave — usa useRouter()
```

- **`page.tsx` (landing)** es solo composición: dos `<Link>`, nada de
  fetching ni estado.
- **`list/page.tsx`** fetchea sus propios datos con `useState` +
  `useEffect` (ver el patrón de fetch más abajo) y se re-fetchea solo
  con volver a montarse — o sea, cada vez que se navega de vuelta acá
  después de crear/editar. No hace falta ningún mecanismo para "avisar"
  que la lista cambió.
- **`create/` y `edit/`** ya no reciben `onClose`/`onCreated`/`onSave`
  por prop ni los leen de Context: llaman `useRouter()` directamente y
  navegan a la lista, tanto al confirmar como al cancelar. El botón
  "Cancelar" reemplaza a la X de cerrar que tenía el modal.
- **`layout.tsx`** es el lugar para guards que antes vivían repetidos en
  cada `page.tsx` (ej: redirigir si el usuario no es admin) — se escribe
  una sola vez y aplica a todo lo que cuelga de esa carpeta.

## 🧍 Componentes autosuficientes

Si un valor no depende de quién esté usando el componente — es el mismo
sin importar el padre, porque sale de algo verdaderamente global (el
usuario logueado, sus roles) — el componente lo resuelve **por su
cuenta**, llamando el hook/service correspondiente, en vez de recibirlo
por prop.

```tsx
// Mal: cada padre tiene que saber calcular esto y pasarlo
<TicketDetail canApprove={canApproveTickets(user)} ticket={ticket} />

// Bien: TicketDetail lo resuelve solo
export default function TicketDetail({ ticket }: TicketDetailProps) {
  const { user } = useCurrentUser();
  const canApprove = canApproveTickets(user);
  ...
}
```

**Cómo saber si aplica:** si cada llamador tendría que escribir la misma
línea (`const isAdmin = checkIsAdmin(user)`) solo para pasarla por prop,
es señal de que el componente debería resolverlo internamente. Si en
cambio el valor sí varía según quién lo usa (ej: `ticket` en sí, que
cada página resuelve distinto), ahí sigue siendo prop normal, padre a
hijo directo.

Esto no reemplaza la regla de props/Context — sigue aplicando para datos
que **sí** son específicos del árbol (ver la sección siguiente). Es
específicamente para datos globales que hoy se resuelven con una llamada
a un hook compartido (`useCurrentUser`, o cualquier otro que cumpla el
mismo rol en el futuro).

## 🔀 Reglas de paso de datos

1. **Props:** solo se usan para pasar datos de un componente a su **hijo
   directo**.
2. **Context (padre → nieto):** si un dato necesita llegar a un
   descendiente que **no** es hijo directo (nieto, bisnieto, etc.), no se
   pasa por props en cadena. Se usa Context.
3. **Context (hijo → padre):** si un componente necesita enviar datos
   hacia un ancestro que no es su padre directo, también se usa Context.

En resumen: **props únicamente para relación padre-hijo directa.**
Cualquier otro intercambio (hacia arriba, o hacia abajo saltando niveles)
se resuelve con Context.

**Antes de aplicar esta regla, primero preguntate si la necesitás.** La
experiencia en este proyecto fue que la mayoría de los casos que
parecían necesitar Context (compartir estado entre una lista y un modal
de crear/editar) dejaron de necesitarlo apenas se convirtieron en rutas
separadas — ver "Rutas en vez de modales" arriba. Reachar para Context
tiene sentido cuando el intercambio es real y no hay forma de
resolverlo con navegación entre páginas.

## 🧩 Reglas de uso de Provider

- Un componente **solo** tiene su propio `Provider` cuando efectivamente
  necesita intercambiar datos con un ancestro o un descendiente no
  directo. No se crea un Provider "por las dudas".
- El `Provider` siempre envuelve al **primer ancestro común** entre el
  emisor y el receptor del dato — el nivel más alto necesario para que
  ambos queden dentro de su árbol.
- **Convención de lectura:** la presencia de `provider`/`context` en una
  carpeta es señal explícita de que ahí hay intercambio de datos no
  directo. Si no hay ese intercambio, esos archivos no deberían existir.
- Cuando haya más de un `Provider` anidado en el mismo árbol, memoizar el
  `value` del Context (`useMemo`) para no generar renders en cascada en
  cada componente que se agrega.
- **Si el Context expone tanto datos como funciones, separalos en dos
  Contexts** (`ComponentNameState` + `ComponentNameActions`), cada uno
  con su propio hook (`useComponentNameState` / `useComponentNameActions`).
  Las funciones (`useCallback` con deps estables) quedan en un objeto
  memoizado que nunca cambia de identidad — un componente que solo
  necesita disparar una acción (no leer estado) nunca vuelve a
  renderizar cuando el estado cambia, porque ni siquiera está
  suscripto a ese Context.
- Un valor mutable simple sin ninguna regla que proteger (ej: abrir/cerrar
  algo) puede exponerse como el setter crudo de `useState`
  (`Dispatch<SetStateAction<T>>`) en vez de envolverlo en una función con
  nombre — envolver tiene sentido cuando la función hace más de una cosa
  a la vez (ej: cerrar **y** registrar qué se aprobó), no por consistencia
  visual.

## 🌐 Carpeta `common/` — código compartido entre páginas

`common/` es **hermana de `app/`**, no vive adentro. Guarda todo lo que no
pertenece a una sola página: api clients, dtos, contexts, hooks o
componentes de UI reutilizados por más de una feature **no relacionada**.

```
app/
common/
├── use-current-user/
│   ├── use-current-user.ts
│   ├── use-current-user.api.ts
│   ├── use-current-user.dto.ts
│   └── use-current-user.service.ts
```

**Misma regla que `components/`:** cada unidad tiene su propia carpeta
kebab-case, con los mismos archivos hermanos (`.tsx`, `.api.ts`,
`.dto.ts`, `.service.ts`, `.provider.tsx`, `.context.ts`, `use-*.ts`),
usando solo los que su responsabilidad requiere. No hay una estructura
distinta para "lo compartido" — es la misma regla, en otro nivel del
árbol.

### Regla de promoción (cuándo algo pasa a `common/`)

1. Todo nace **local**, dentro de `components/` de la página o del
   componente que lo necesita por primera vez.
2. Se promueve a `common/` recién cuando aparece un **segundo
   consumidor** que no comparte el mismo subárbol — otra página, u otro
   componente fuera del árbol del primero.
3. Nunca se crea algo directamente en `common/` "por las dudas" o
   anticipando reuso futuro. Si hoy lo usa un solo componente, vive ahí.

**Matiz sobre "quién promueve a `common/` vs. quién promueve solo un
nivel":** si el segundo consumidor sigue estando dentro de la misma
familia de rutas (ej: `list/` y `[id]/edit/`, ambos bajo
`app/home/users/`), lo compartido se promueve al ancestro común de esa
familia (`app/home/users/users.dto.ts`), no hasta `common/`. `common/`
es para cuando el segundo consumidor está en otra parte del árbol que
no comparte ningún ancestro razonable — por ejemplo `common/use-current-user`,
que usan páginas de features completamente distintas.

Esto puede significar que un tipo lo consuma un componente fuera de la
feature que lo define (ej: `HomeHeader`, fuera de `app/home/tickets/`,
importa `TicketDetails` desde `app/home/tickets/tickets.dto.ts`) y aun
así se decida dejarlo ahí en vez de subirlo a `common/`, si el tipo
sigue siendo conceptualmente del dominio de esa feature. Es una decisión
de criterio, no una regla mecánica — cuando dude, preguntar.

### Import

Se importa siempre con el alias absoluto del proyecto
(`@/common/auth-context/...` o `@/app/home/users/users.dto`, según
dónde haya quedado), nunca con rutas relativas largas
(`../../../common/...`).

## ⚠️ Fetch en efectos: patrón obligatorio

Este proyecto tiene `react-hooks/set-state-in-effect` activo en el
lint — **falla el build**, no es un warning cosmético. La regla: nunca
llamar a un setter de estado de forma síncrona dentro del cuerpo de un
`useEffect`. Solo está permitido dentro de un `.then()`/`.catch()` de
una llamada async.

```tsx
// Correcto — el único patrón usado en todo el proyecto
useEffect(() => {
  let cancelled = false;

  fetchThing()
    .then((data) => {
      if (cancelled) return;
      setData(data);
      setIsLoading(false);
    })
    .catch(() => {
      if (!cancelled) {
        setError(LOAD_ERROR_MESSAGE);
        setIsLoading(false);
      }
    });

  return () => {
    cancelled = true;
  };
}, [/* deps que deberían disparar un refetch */]);
```

Si necesitás que un componente reaccione a que otra cosa cambió en otro
lado del árbol (ej: refrescar una lista porque se aprobó algo en otra
página), no agregues un `useEffect` nuevo que llame `setState`
directamente — sumá esa dependencia al **mismo** effect que ya hace el
fetch envuelto en `.then()/.catch()`, en vez de un effect separado con
`setState` suelto en el cuerpo.

## 📌 Resumen de reglas

- `page.tsx` = estructura + composición, sin lógica. La landing de un
  flujo con varias acciones (crear, ver, listar) es solo un puñado de
  `<Link>`.
- Preferí una ruta separada (`list/`, `create/`, `[id]/...`) antes que
  un modal + estado compartido con la lista — resuelve solo casi todos
  los casos que antes pedían Context.
- Un componente que necesita un valor global (rol del usuario, etc.) lo
  resuelve solo, llamando el hook/service compartido, en vez de
  recibirlo por prop de cada padre posible.
- Cada unidad (componente o pieza de `common/`) vive en su propia carpeta
  con archivos hermanos por responsabilidad, creando solo los que
  necesita.
- Props → solo padre-hijo directo. Context → solo cuando de verdad no
  hay forma de resolverlo con rutas o con "componente autosuficiente".
- Provider → solo donde realmente hay intercambio de datos, envolviendo
  el ancestro común más alto necesario; separar `State`/`Actions` en dos
  Contexts si hay ambos; memoizar `value`.
- `common/` → para lo que cruza features no relacionadas. Si el segundo
  consumidor sigue dentro de la misma familia de rutas, promocioná solo
  hasta el ancestro común de esa familia, no hasta `common/`.
- Todo fetch en un `useEffect` sigue el patrón `cancelled` +
  `setState` solo dentro de `.then()/.catch()` — nunca síncrono en el
  cuerpo del effect.
