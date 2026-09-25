# Análisis de amenazas — blog-ai — 2026-09-25

- **Fecha:** 2026-09-25 (UTC)
- **Commit:** 0df7edc
- **Alcance:** El servicio de RAG `blog-ai` (FastAPI + pgvector + Ollama) y el harness de Claude Code que hay en este repositorio, en la rama `s7/start`.

> Nota: la skill está escrita para FlowSync (`backend/`, `frontend/`, `package.json`). Este repositorio no tiene esas carpetas: la superficie «producto» se ha aplicado a `app/`, `db/` y `scripts/`, que es donde vive el código de `blog-ai`.

## Superficies

### Producto
Sí integra modelos de lenguaje, locales, vía HTTP a Ollama (`OLLAMA_URL`, por defecto `http://localhost:11434`), sin SDK: `httpx` está en `requirements.txt` y no hay `anthropic`, `openai` ni `langchain`. `app/servicios/embeddings.py` llama a `/api/embed` con `nomic-embed-text:latest`; `app/servicios/generacion.py` llama a `/api/chat` con `qwen2.5:3b-instruct`, con un system prompt fijo (`INSTRUCCIONES`), los fragmentos recuperados y la pregunta del lector en el mensaje de usuario, sin `tools`. Los vectores van a la tabla `fragmentos` de PostgreSQL/pgvector (`app/base_de_datos.py`). Hay tres rutas que tocan el modelo, todas sin autenticación: `POST /indexar`, `POST /buscar` y `POST /preguntar` (`app/main.py`). `make up` arranca uvicorn sin `--host`, así que escucha en `127.0.0.1:8402` (`Makefile`).

### Harness
Solo existe `.claude/skills/analisis-de-amenazas/SKILL.md` (sin trackear en git). No hay `CLAUDE.md`, `AGENTS.md`, `.claude/settings.json`, `.claude/settings.local.json`, `.claude/hooks/`, `.claude/agents/`, `.claude/commands/`, `.mcp.json` ni `.github/workflows/`: se buscaron y no están.

### Archivos abiertos
- `.claude/skills/analisis-de-amenazas/SKILL.md`
- `README.md`
- `requirements.txt`
- `.env.example`
- `.gitignore`
- `docker-compose.yml`
- `Makefile`
- `app/__init__.py`
- `app/main.py`
- `app/esquemas.py`
- `app/configuracion.py`
- `app/base_de_datos.py`
- `app/servicios/__init__.py`
- `app/servicios/embeddings.py`
- `app/servicios/generacion.py`
- `app/servicios/indice.py`
- `db/init.sql`
- `scripts/generar_openapi.py`

## LLM01 — Prompt Injection
**Qué se miró:** `app/servicios/generacion.py`, `app/main.py`, `app/esquemas.py`, `app/servicios/indice.py`, `Makefile`
**Decisión:** `aceptado` — La pregunta del lector (`PeticionPreguntar.consulta`, hasta 500 caracteres) y el texto de los posts indexados entran sin delimitar en el mensaje de usuario que monta `responder()`, y `/indexar` acepta contenido de cualquiera que alcance el puerto; se acepta porque la llamada a `/api/chat` no declara `tools`, la salida es solo texto devuelto en JSON y `make up` escucha en `127.0.0.1`. Lo volvería inaceptable que el modelo reciba herramientas o acciones, que el servicio se exponga fuera de localhost, o que se indexe contenido escrito por usuarios no editores (comentarios, posts de terceros).

## LLM02 — Sensitive Information Disclosure
**Qué se miró:** `app/servicios/indice.py`, `app/esquemas.py`, `app/servicios/generacion.py`, `app/main.py`, `.env.example`, `.gitignore`
**Decisión:** `mitigado` — Los borradores nunca llegan al contexto del modelo: `PostEntrante.estado` es `Literal["borrador", "publicado"]` en `app/esquemas.py` e `indexar()` en `app/servicios/indice.py` los omite y además borra sus fragmentos con `_borrar_post()`, así que el índice solo contiene texto ya publicado; el prompt no incluye credenciales (salen de `.env`, ignorado en `.gitignore`), y lo único interno que se filtra son los 200 primeros caracteres de un error de Ollama en el 503 de `app/main.py`.

## LLM03 — Supply Chain
**Qué se miró:** `requirements.txt`, `Makefile`, `app/configuracion.py`, `.env.example`, `docker-compose.yml`, `.claude/skills/analisis-de-amenazas/SKILL.md`
**Decisión:** `aceptado` — Las dependencias Python están fijadas con `==` en `requirements.txt`, pero los modelos se descargan por etiqueta mutable (`nomic-embed-text:latest`, `qwen2.5:3b-instruct` en `Makefile` y `app/configuracion.py`) sin digest, y la imagen `pgvector/pgvector:pg16` de `docker-compose.yml` tampoco va fijada; la única pieza del harness es una skill local sin dependencias externas. Se acepta por ser un entorno de curso en local; lo volvería inaceptable desplegarlo en un servidor compartido o que `make modelos` apunte a un registro que no sea el oficial de Ollama.

## LLM04 — Data and Model Poisoning
**Qué se miró:** `app/main.py`, `app/servicios/indice.py`, `app/esquemas.py`, `Makefile`, `README.md`
**Decisión:** `aceptado` — No hay entrenamiento ni ajuste fino, pero la base de conocimiento del RAG sí se puede envenenar: `POST /indexar` no tiene autenticación (lo reconoce `README.md`) y sobrescribe los fragmentos de cualquier `post_id` que se le mande. Se acepta porque `make up` escucha solo en `127.0.0.1` y el único cliente previsto es `blog-api`; lo volvería inaceptable arrancar con `--host 0.0.0.0`, desplegarlo o que otro proceso distinto de `blog-api` pueda alcanzar el puerto 8402.

## LLM05 — Improper Output Handling
**Qué se miró:** `app/servicios/generacion.py`, `app/main.py`, `app/esquemas.py`
**Decisión:** `fuera de alcance` — En este repositorio la salida del modelo no se ejecuta, no entra en SQL ni se renderiza: `responder()` hace `.strip()` y `/preguntar` la devuelve como `RespuestaPreguntar.respuesta` (string en JSON). Dónde se interpreta (HTML en `blog-web`, vía `blog-api`) está en otros repositorios que no se abrieron. Se reabre cuando `blog-ai` use la respuesta para algo más que devolverla (guardarla, construir consultas, llamar a otro servicio) o cuando el análisis cubra `blog-api`/`blog-web`.

## LLM06 — Excessive Agency
**Qué se miró:** `app/servicios/generacion.py`, `.claude/skills/analisis-de-amenazas/SKILL.md`
**Decisión:** `aceptado` — En el producto el modelo no tiene agencia: `/api/chat` se llama sin `tools` y el código no actúa según lo que responde. En el harness no hay `.claude/settings.json`, hooks, agentes, comandos ni `.mcp.json`, así que el repositorio no restringe ni amplía lo que puede hacer el agente de desarrollo: sus permisos dependen de la configuración global de cada máquina, y la única skill solo escribe en `docs/seguridad/`. Lo volvería inaceptable dar herramientas al modelo de generación, o añadir al harness un MCP, hook o workflow con acceso a secretos, a la base de datos o a `git push`.

## LLM07 — System Prompt Leakage
**Qué se miró:** `app/servicios/generacion.py`
**Decisión:** `aceptado` — `INSTRUCCIONES` solo contiene el idioma, la longitud, la forma de citar y la frase de «no lo encuentro»; que se filtre no expone secretos ni reglas de autorización. Lo volvería inaceptable meter en el system prompt credenciales, URLs internas o reglas de control de acceso.

## LLM08 — Vector and Embedding Weaknesses
**Qué se miró:** `app/base_de_datos.py`, `app/servicios/indice.py`, `app/servicios/embeddings.py`, `app/configuracion.py`, `README.md`
**Decisión:** `aceptado` — La tabla `fragmentos` es de un solo inquilino y sin control de acceso por fila, y `buscar()` devuelve cualquier fragmento; se acepta porque solo contiene posts publicados (ver LLM02). El riesgo de integridad conocido es cambiar `MODELO_EMBEDDINGS` sin reindexar, que el `README.md` documenta que degrada la búsqueda sin error; la dimensión sí se controla con `VECTOR({dimension_embedding})`. Lo volvería inaceptable indexar contenido privado o de varios usuarios en la misma tabla, o que el cambio de modelo de embeddings se automatice sin reindexado.

## LLM09 — Misinformation
**Qué se miró:** `app/servicios/generacion.py`, `app/servicios/indice.py`, `app/main.py`, `README.md`
**Decisión:** `aceptado` — `responder()` usa `temperature: 0.2`, obliga a citar `[n]` y a decir «No lo encuentro…» y `/preguntar` devuelve las `fuentes` con su puntuación; pero `buscar()` no aplica umbral mínimo de puntuación, así que el modelo de 3B recibe fragmentos poco relacionados y, como admite `README.md`, «los une igual» y responde con seguridad algo equivocado. Se acepta para el ejercicio; lo volvería inaceptable publicar `/preguntar` a lectores reales sin umbral de puntuación y sin mostrar las fuentes junto a la respuesta.

## LLM10 — Unbounded Consumption
**Qué se miró:** `app/esquemas.py`, `app/servicios/generacion.py`, `app/servicios/embeddings.py`, `app/servicios/indice.py`, `app/base_de_datos.py`, `Makefile`
**Decisión:** `aceptado` — `/buscar` y `/preguntar` están acotados (`consulta` ≤ 500 caracteres, `limite` ≤ 20 y ≤ 10 en `app/esquemas.py`, `num_predict: 320` y timeouts de 120/180 s en los clientes `httpx`), pero `/indexar` no limita el número de posts ni el tamaño de `cuerpo`, no hay rate limiting ni autenticación, y la conexión única síncrona de `app/base_de_datos.py` bloquea el bucle de eventos. Se acepta porque escucha en `127.0.0.1` y los modelos son locales (sin coste por token); lo volvería inaceptable exponer el puerto 8402 o sustituir Ollama por una API de pago.

## Fuera de alcance y cuándo se reabre
| Entrada | Motivo | Disparador para reabrir |
|---|---|---|
| LLM05 | `blog-ai` solo devuelve la respuesta del modelo como string en JSON; quien la interpreta o renderiza está en `blog-api`/`blog-web`, fuera de este repositorio. | Cuando `blog-ai` use la respuesta para algo más que devolverla, o cuando el análisis cubra `blog-api`/`blog-web`. |
