# Equipo de agentes — Deber 6

| Agente | Misión | Herramientas autorizadas | Entrada | Salida | Límites |
|---|---|---|---|---|---|
| Orquestador | repartir la revisión de `usuarios.py` entre los dos especialistas y consolidar sus resultados | ninguna propia (solo delega) | ruta del archivo: `usuarios.py` | tabla con hallazgos de seguridad y casos de prueba propuestos | no modifica archivos |
| Especialista 1 — Seguridad | buscar en `usuarios.py` claves escritas en el código (constantes como `API_KEY`, contraseñas) y entradas sin validar (parámetros de funciones usados directamente en consultas SQL o comparaciones) | leer, buscar | ruta del archivo: `usuarios.py` | lista: línea, problema, severidad | solo `usuarios.py`; no escribe |
| Especialista 2 — Pruebas | proponer 3 casos de prueba que faltan para las funciones de `usuarios.py` (`buscar_usuario`, `calcular_descuento`, `crear_usuario`, `es_admin`) | leer | ruta del archivo: `usuarios.py` | lista: nombre del caso y qué verifica | solo propone; no escribe pruebas |


## 1. Diagrama del flujo

Flujo paralelo: los dos especialistas no dependen uno del otro. El checkpoint humano va al final, después de que el orquestador consolida la tabla y antes de darla por buena.

```
                usuarios.py
                     │
                Orquestador (reparte)
                ┌────┴────┐
                ▼         ▼       
         Seguridad      Pruebas
                └────┬────┘
                     ▼
          Orquestador (valida y consolida)
                     │
            CHECKPOINT HUMANO
                     │
                Tabla final
```

## 2. Qué mira la persona antes de aprobar

Antes de aprobar, la persona abre `usuarios.py` junto a la tabla y revisa:

Que cada hallazgo exista de verdad que la línea indicada tenga el problema descrito.
Que la severidad tenga sentido: una clave escrita en el código o una inyección SQL no puede quedar como "baja".
Que los casos de prueba sean útiles: que sean exactamente 3, que prueben funciones que existen en `usuarios.py`, que digan qué verifican y que no repitan lo mismo con otras palabras.

La persona rechaza si:

aparece un hallazgo que no existe en el código, o con un número de línea equivocado;
un caso de prueba no tiene sentido;
falta un problema evidente de seguridad;
algún agente modificó un archivo (se rompió un límite).

Al rechazar, la persona dice qué está mal y el orquestador vuelve a pedir la corrección al especialista correspondiente.

## 3. Cómo se detecta que un especialista falló y qué hace el orquestador

| Falla | Cómo se detecta | Qué hace el orquestador |
|---|---|---|
| Salida vacía | el especialista no devuelve ningún hallazgo o caso de prueba | reintenta una vez con la misma instrucción; si vuelve a salir vacía, avisa a la persona |
| Inventa líneas | el número de línea no existe en `usuarios.py` (el archivo tiene 43 líneas) o el texto de esa línea no coincide con el problema | pide corrección indicando qué línea no cuadra; si lo repite, descarta ese hallazgo y avisa a la persona |
| Formato incorrecto | falta línea, problema o severidad (Seguridad), o falta nombre o qué verifica (Pruebas), o no son 3 casos | pide corrección con el formato exacto que se espera |
| Ignora un límite | revisa otro archivo, intenta escribir o modificar archivos, o escribe código de pruebas en vez de solo proponer | descarta esa salida, no la incluye en la tabla y avisa a la persona |
| Sale del tema | Seguridad propone pruebas o Pruebas reporta problemas de seguridad | ignora lo que no le toca y, si no queda nada útil, pide corrección |

Regla general: máximo un reintento por especialista.

## A2A

Supuesto: el Especialista 2 (Pruebas) ahora pertenece a otra empresa y corre en su propio servidor. El orquestador ya no lo tiene como subagente local: tiene que descubrirlo y encargarle la tarea mediante A2A. Esto es solo diseño; no hay servidor, endpoint real, credenciales ni tráfico HTTP.

### 1. Agent Card ilustrativa

> Simplificación didáctica; no es JSON conforme a una versión específica de A2A.

```json
{
  "name": "Agente de Pruebas",
  "description": "Propone casos de prueba que faltan para las funciones de un archivo Python. Solo propone; no escribe ni ejecuta código.",
  "version": "1.0.0",
  "provider": {
    "organization": "Empresa Ejemplo de QA",
    "url": "https://agente.ejemplo"
  },
  "url": "https://agente.ejemplo/a2a",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["text/plain"],
  "skills": [
    {
      "id": "proponer-casos-prueba",
      "name": "Proponer casos de prueba",
      "description": "Recibe el código de un archivo Python y devuelve 3 casos de prueba que faltan, cada uno con su nombre y qué verifica.",
      "tags": ["pruebas", "python", "calidad"],
      "examples": ["Propón 3 casos de prueba que faltan para usuarios.py"]
    }
  ],
  "securitySchemes": {
    "bearerAuth": {
      "type": "http",
      "scheme": "bearer"
    }
  },
  "security": [{ "bearerAuth": [] }]
}
```

La Agent Card solo **anuncia** quién es el agente, qué skill ofrece, dónde recibe solicitudes (`url`) y qué seguridad exige (`bearerAuth`). No ejecuta tareas ni prueba que el agente sea confiable.

### 2. Dónde se obtiene la tarjeta y qué valida el cliente

Dónde se obtiene:
- En la ruta estándar `https://agente.ejemplo/.well-known/agent-card.json`.
- O en una URL que la otra empresa nos entrega directamente, o en un registro de agentes de la organización.

Qué valida el orquestador antes de usarla:
- **Origen:** que la tarjeta venga del dominio esperado (`agente.ejemplo`) y por HTTPS.
- **Skill adecuada:** que exista la skill `proponer-casos-prueba` y que su descripción coincida con lo que necesitamos (3 casos de prueba, solo proponer).
- **Interfaz:** que el endpoint (`url`) y los formatos de entrada/salida (`text/plain`) sean compatibles con nuestro cliente.
- **Autenticación requerida:** que exige un token Bearer y que tenemos una credencial válida de ese tipo. Sin ella no se envía nada.


### 3. Secuencia

```
Orquestador                                   Agente de Pruebas (agente.ejemplo)
    │                                                   │
    │ 1. GET /.well-known/agent-card.json ────────────► │
    │ ◄──────────────────────────────── Agent Card      │
    │                                                   │
    │ 2. Verificar skill, interfaz y seguridad          │
    │                                                   │
    │ 3. Enviar mensaje (código de usuarios.py          │
    │    + "propón 3 casos de prueba") ───────────────► │
    │                                                   │
    │ ◄──── 4a. Respuesta inmediata (mensaje), o        │
    │ ◄──── 4b. Task creada (estado: submitted)         │
    │                                                   │
    │ 5. Consultar / recibir estado (working...) ◄────► │
    │                                                   │
    │ ◄──── 6. completed + resultado (Artifact)         │
    │                                                   │
    │ 7. Validar el resultado y pasarlo al checkpoint humano
```

Una solicitud A2A **no siempre crea una Task**: si la tarea es simple, el agente remoto puede responder directamente con un mensaje (4a) y la secuencia termina ahí. Si la tarea toma tiempo, crea una Task (4b) y el orquestador sigue su estado hasta obtener el Artifact.

### 4. Estados

Camino simplificado:
- `submitted`: el agente remoto recibió la tarea y la aceptó.
- `working`: el agente está analizando el código y preparando los casos de prueba.
- `completed`: la tarea terminó y el resultado (los 3 casos de prueba) está disponible como Artifact.

Caso alternativo `input-required`: el agente necesita más información para continuar, por ejemplo porque recibió solo el nombre del archivo y no su contenido. El orquestador envía lo que falta (el código de `usuarios.py`) en la misma Task. Si no sabe qué responder, pregunta a la persona antes de continuar.

Caso alternativo `failed`: el agente no pudo completar la tarea. El orquestador reintenta una vez (igual que la regla general de la sección 3). Si vuelve a fallar, no incluye casos de prueba en la tabla y avisa a la persona.

### 5. Qué cambia frente al subagente local

| Aspecto | Subagente local (Claude Code) | Agente remoto A2A |
|---|---|---|
| Interfaz de red | no hay red; el harness lo ejecuta dentro de la misma sesión | HTTP(S) hacia el endpoint de `agente.ejemplo` |
| Descubrimiento | archivo en `.claude/agents/`; el harness lo lee | se descubre leyendo su Agent Card |
| Interoperabilidad | solo funciona dentro de Claude Code | cualquier cliente compatible con A2A, de cualquier proveedor |
| Permisos | los controla el harness con la línea `tools:` (solo leer) | no los controlamos; solo vemos lo que anuncia la tarjeta y exigimos autenticación |
| Límites de confianza | es nuestro y corre en nuestra máquina | es un tercero: no le enviamos secretos, validamos su respuesta y descubrirlo no implica confiar en él |
