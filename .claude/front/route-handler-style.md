# Estilo de código para Route Handlers (`app/api/**/route.ts`)

Flujo lineal con guard clauses y nombres explícitos. Evolución de la
descomposición vertical: además de aplanar el control de flujo, define
cómo nombrar y organizar todo lo que se extrae.

Aplica a cualquier handler HTTP (`GET`, `POST`, `PUT`, `DELETE`, ...)
dentro de un Route Handler de Next.js.

## 🎯 Principio

El handler exportado se lee como una lista de validaciones y pasos, cada
uno con `return` temprano ante el primer problema. Nada del "cómo" vive
en el handler — todo el detalle técnico se extrae a funciones con nombre
de intención, y todo valor mágico (status, mensaje, timeout, límites,
opciones repetidas) se sube a una constante o función nombrada.

## ✅ Reglas

1. **Guard clauses, nunca `if/else` anidado.** Cada validación corta el
   flujo con `return` inmediato. El handler nunca baja de un nivel de
   indentación — si una rama necesita más lógica, esa lógica se extrae a
   su propia función.

2. **Resultado explícito en vez de excepción para fallos esperados.**
   Cualquier función que pueda fallar de forma esperada (parsear el
   body, llamar a un servicio externo, leer un recurso) atrapa su propio
   `try/catch` internamente y devuelve `null` (o un tipo `T | null` /
   `{ ok: boolean }`) — el llamador nunca maneja excepciones, solo
   chequea `if (!resultado)`. El `catch` interno siempre loguea el error
   con contexto (`console.error("Failed to ...", error)`) antes de
   devolver el resultado vacío; el error nunca se pierde en silencio.

3. **Predicados con nombre en vez de condiciones inline.** Nunca una
   comparación compleja (rangos de status, validación de forma de un
   objeto, chequeos de negocio) suelta en medio del flujo — se extrae a
   una función `isAlgo(x)` / `hasAlgo(x)`. Cuando el predicado también
   sirve para angostar un tipo, se escribe como *type guard*
   (`x is TipoEspecifico`) en vez de devolver `boolean` a secas.

4. **Constantes arriba, agrupadas por rol, con sufijo que indica su
   forma.** Por ejemplo: sufijo para objetos `{ status }` listos para
   pasarle al framework, sufijo distinto para los bodies de respuesta,
   sufijo distinto para configuración (timeouts, límites, nombres). El
   sufijo le dice al lector qué es la constante sin que tenga que leer
   su valor.

5. **Cada `return` de error arma su propia respuesta con sus propias
   constantes**, sin una función centralizadora tipo
   `buildErrorResponse(status)` que oculte cuál mensaje corresponde a
   cuál caso. Se prioriza que cada línea sea autoexplicativa por sobre
   eliminar la repetición de la llamada al framework.

6. **Objetos de configuración repetidos van en una función `buildX()`**,
   no como objeto literal inline en el punto de uso — sobre todo si el
   objeto depende de variables de entorno o se arma en más de un lugar.

7. **Orden de las funciones: el handler exportado va primero, sus
   colaboradoras debajo**, más o menos en el orden en que se usan. El
   lector nunca tiene que bajar a leer el detalle antes de entender el
   flujo general.

8. **Nombre = intención del dominio, no la mecánica.** Ej.:
   `loginWithBackend`, no `fetchAuthEndpointAndParseJson`;
   `buildCookieOptions`, no `getCookieConfigObject`.

9. **Llamadas a servicios externos (`fetch`, DB, etc.) siempre con
   timeout explícito** (`AbortSignal.timeout(...)`), nunca sin límite —
   un handler no puede quedar esperando para siempre a una dependencia
   caída.

## 📌 Ejemplo genérico

```ts
export async function POST(request: NextRequest) {
  const input = await parseRequestBody(request);
  if (!input) return NextResponse.json(GENERIC_ERROR_MESSAGE, BAD_REQUEST_STATUS);

  if (!hasValidShape(input)) {
    return NextResponse.json(GENERIC_ERROR_MESSAGE, BAD_REQUEST_STATUS);
  }

  const result = await callExternalService(input);
  if (!result) {
    return NextResponse.json(SERVICE_UNREACHABLE_MESSAGE, SERVICE_UNAVAILABLE_STATUS);
  }

  if (isClientError(result)) {
    return NextResponse.json(GENERIC_ERROR_MESSAGE, UNAUTHORIZED_STATUS);
  }

  if (isServerError(result)) {
    return NextResponse.json(SERVICE_ERROR_MESSAGE, SERVICE_UNAVAILABLE_STATUS);
  }

  return NextResponse.json({ ok: true });
}

async function parseRequestBody(request: NextRequest): Promise<Input | null> {
  try {
    return await request.json();
  } catch (error) {
    console.error("Failed to parse request body", error);
    return null; // el llamador nunca ve la excepción
  }
}

async function callExternalService(input: Input): Promise<Response | null> {
  try {
    return await fetch(EXTERNAL_URL, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(input),
      signal: AbortSignal.timeout(REQUEST_TIMEOUT_MS),
    });
  } catch (error) {
    console.error("Failed to reach external service", error);
    return null;
  }
}

function isClientError(response: Response): boolean {
  return response.status >= 400 && response.status < 500;
}

function isServerError(response: Response): boolean {
  return response.status >= 500;
}

function hasValidShape(input: Input): input is ValidInput {
  return typeof input.someField === "string";
}
```
