# Proyecto Final — Asistente RAG de Tickets de Soporte de Datos

**Asignatura:** Desarrollo de IA · Profesor: Ricardo (`rpmaya`)
**Autor:** Diego Martín

Sistema RAG (Retrieval-Augmented Generation) construido sobre un corpus de
**tickets de soporte de datos** documentados (problema → diagnóstico → actuación →
resolución). Permite hacer preguntas en lenguaje natural sobre cómo se resolvieron
incidencias anteriores, recuperando los tickets más relevantes y respondiendo
**exclusivamente** a partir de ellos.

> ⚠️ **Datos sanitizados.** Todo el corpus es ficticio (empresa inventada
> *"Conexia Telecom"*, roles realistas pero no reales, IDs y clientes inventados).
> No contiene información de ningún sistema o cliente real.

---

## 1. Motivación

En soporte de datos muchas incidencias se repiten con variaciones (el dato falla
en el origen, en la transformación, en el refresco o en el modelo). Tener un
asistente que recupere "¿cómo resolvimos antes un problema parecido?" ahorra
tiempo de diagnóstico y conserva el conocimiento de los tickets ya cerrados.

El corpus refleja la variedad real de un equipo de soporte: tickets **resueltos**
en el propio equipo, **escalados** a Ingeniería de Datos, y **reasignados** a otros
equipos (permisos, configuración de Power BI) cuando no eran de datos.

## 2. Arquitectura

```
                    INGESTA (una vez)
  8 tickets .docx (Google Drive)
        │  Docx2txtLoader
        ▼
   Troceado (chunk_size=1000, overlap=150)
        │  OpenAIEmbeddings (text-embedding-ada-002)
        ▼
   ChromaDB  (base vectorial local, persist_directory)

                    CONSULTA (por pregunta)
  Pregunta ──► embedding ──► búsqueda por similitud (k=6)
        │
        ▼  contexto recuperado (chunks + nombre de fichero fuente)
   Prompt + contexto ──► gpt-4o-mini (temperature=0) ──► Respuesta
```

El LLM tiene instrucción explícita de responder **solo** con la información del
contexto y de decir *"no consta"* cuando la respuesta no está en los tickets, lo
que evita alucinaciones (ver caso negativo en los resultados).

## 3. Cómo cubre las unidades

| Unidad | Componente del proyecto |
|--------|-------------------------|
| **U1 — LLM** | `gpt-4o-mini` como motor de generación |
| **U2 — Prompts** | Prompt con instrucción anti-alucinación e inyección de contexto |
| **U3 — API** | API de OpenAI (embeddings + chat) con clave vía `getpass` |
| **U5 — RAG** | Pipeline completo LangChain + ChromaDB |
| **U6 — Conector** *(en curso)* | Servidor MCP que expone la base de tickets a Claude Desktop |

## 4. Decisiones técnicas

| Decisión | Elección | Motivo |
|----------|----------|--------|
| Base vectorial | **ChromaDB** (local) | Reproducible desde el repo, sin coste, encaja con el conector MCP local |
| LLM | **gpt-4o-mini**, `temp=0` | Barato, suficiente para Q&A factual; temperatura 0 para respuestas estables |
| Embeddings | **text-embedding-ada-002** | El usado en clase, consistente con las prácticas previas |
| Tamaño de chunk | **1000 / overlap 150** | Cada ticket es corto; así no se separa el diagnóstico de su resolución |
| `k` del retriever | **6** | Cada ticket genera pocos chunks; subir `k` mejora la recuperación de respuestas que abarcan varios tickets |
| Loader | **Docx2txtLoader** | El corpus está en `.docx` (plantilla estructurada por ticket) |

## 5. Corpus de tickets

8 tickets ficticios con resolución variada (esta variedad es lo que permite
evaluar si el RAG distingue bien los tipos de caso):

| Ticket | Tema | Resolución |
|--------|------|------------|
| 001 | KAM no aparece en Booking to Billing | Escalado a Ing. de Datos |
| 002 | Cambios de CRM no se reflejan | Reasignado (refresco de datasets) |
| 003 | `AvailableForBilling = 0` (Colombia) | Workaround + escalado |
| 004 | Versión antigua del booking | Workaround + escalado |
| 005 | Usuario sin acceso a informe | Reasignado (permisos — **no es de datos**) |
| 006 | Totales inflados en un visual | Reasignado (config. Power BI — **no es de datos**) |
| 007 | Faltan ventas FTTH | **Resuelto** (inserción manual transaccional) |
| 008 | Dataset no refresca | **Resuelto** (fichero bloqueado en SharePoint) |

## 6. Resultados de las pruebas

Pruebas ejecutadas sobre el sistema ya montado. Se incluyen tanto los aciertos
como los fallos, con su análisis.

| # | Pregunta | Respuesta del sistema | ¿Correcta? |
|---|----------|----------------------|------------|
| 1 | ¿Qué tickets resolví **yo** directamente y cómo? | "No consta." | ❌ (esperado: 007 y 008) |
| 2 | ¿Cuáles se escalaron a Ingeniería de Datos y por qué? | "No consta." | ❌ (esperado: 001, 003, 004) |
| 3 | ¿Qué tickets NO eran de datos? | TICKET-005 (Permisos) | ✅ |
| 4 | Resume el TICKET-001 y su causa raíz | Resume bien el caso (KAM en blanco pese a estar en CRM) y que se escaló a Data Engineering | ✅ |
| 5 | ¿Hay algún ticket sobre el menú de la cafetería? | "No consta." | ✅ **(caso negativo clave)** |
| 6 | ¿Qué tickets resolvió directamente **Soporte de Datos**? *(reformulación de la P1)* | Lista 001/003/004/006 | ⚠️ Parcial (mezcla escalados/reasignados con resueltos; no recupera 007/008) |

El **caso negativo** (pregunta 5) es el más importante de la rúbrica: el sistema
responde *"no consta"* en lugar de inventarse una respuesta, demostrando que se
ciñe al contexto recuperado.

## 7. Limitaciones detectadas y mejoras propuestas

El análisis de los fallos (preguntas 1, 2 y 6) revela tres limitaciones del
sistema en su versión actual:

1. **Desajuste de persona gramatical.** Las preguntas en primera persona
   ("resolví *yo*") no enganchan bien con el texto del corpus, escrito en tercera
   persona ("resuelto por Soporte de Datos"). La similitud semántica baja y el
   sistema devuelve "no consta".
   → *Mejora:* normalizar la pregunta (reescritura/query expansion) o redactar el
   corpus en primera persona.

2. **Confusión entre "actuar sobre" y "resolver".** En la pregunta 6 el modelo
   mete tickets escalados/reasignados (001, 003, 004, 006) bajo "resueltos", porque
   en todos ellos hubo diagnóstico y actuación. Los chunks son semánticamente
   parecidos y el retriever no separa bien el **estado final** del ticket.
   → *Mejora:* añadir el estado de resolución a los metadatos de cada chunk y
   filtrar por metadato, no solo por similitud semántica.

3. **Sensibilidad al parámetro `k`.** Con `k=4` algunas preguntas se respondían y
   con `k=6` cambiaban; la recuperación no es estable.
   → *Mejora:* evaluar con métricas (RAGAS) y aplicar reranking.

**Mejoras futuras (fuera del alcance actual):**
- Conector **MCP** para consultar la base desde Claude Desktop (U6).
- Generación automática de borradores de resolución para tickets nuevos.
- Comparativa **Vector RAG vs GraphRAG (Neo4j)** sobre el mismo corpus.

## 8. Estructura del repositorio

```
PF-IA-Ricardo/
├── README.md                     ← este documento
├── .gitignore                    ← ignora .env, chroma_*/, __pycache__/
├── requirements.txt
├── notebooks/
│   └── ProyectoFinal_RAG.ipynb   ← pipeline completo (ingesta + consulta + pruebas)
└── corpus/
    └── TICKET-001..008_*.docx    ← tickets ficticios sanitizados
```

## 9. Cómo ejecutar

1. Crear un `.env` a partir de `.env.example` con tu `OPENAI_API_KEY`
   (nunca subir el `.env` real al repo).
2. `pip install -r requirements.txt`
3. Abrir el notebook y ejecutar las celdas en orden: dependencias → montar
   Drive/clave → cargar `.docx` → trocear → embeddings/ChromaDB → cadena RAG →
   pruebas.

---

*Proyecto académico con datos ficticios. No contiene información real de ninguna
empresa ni cliente.*
