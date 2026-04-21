# H-Mantis
MCP connector to evaluate a synthetic health data space

# Guía: Cómo conectarse al Espacio de Datos de Salud EHDS
**Proyecto H-Mantis / OEG-UPM**  
**Versión:** Abril 2026

---

## ¿Qué es esto?

Hemos creado un servidor público que permite a cualquier asistente de inteligencia artificial (IA) consultar datos de salud sintéticos. Los datos están organizados según el estándar europeo EHDS (Espacio Europeo de Datos de Salud).

El servidor está disponible en:
```
https://caad-138-100-11-141.ngrok-free.app/sse
```

Puedes hacer preguntas como:
- "¿Qué conjuntos de datos de salud están disponibles?"
- "¿Cuál es el medicamento más común en el grupo de diabetes?"
- "¿Puedo usar estos datos para entrenar un modelo comercial de IA?"

---

## Opción 1: Claude.ai (la más fácil)

**Requisitos:** Cuenta en claude.ai (gratuita o de pago)

**Pasos:**

1. Abre [claude.ai](https://claude.ai) en tu navegador.

2. Haz clic en tu foto de perfil (arriba a la derecha) → **Settings** (Configuración).

3. En el menú de la izquierda, haz clic en **Connectors** (Conectores).

4. Haz clic en **Add custom connector** (Añadir conector personalizado).

5. En el campo de URL, escribe:
   ```
   https://caad-138-100-11-141.ngrok-free.app/sse
   ```

6. Dale un nombre, por ejemplo: **EHDS Health Data Space**

7. Haz clic en **Save** (Guardar).

8. Abre una nueva conversación y pregunta:
   ```
   ¿Qué conjuntos de datos de salud hay disponibles?
   ```

Claude buscará automáticamente en los datos reales y te dará una respuesta precisa.

---

## Opción 2: API de Anthropic (Claude por código)

**Requisitos:** Clave API de Anthropic (`ANTHROPIC_API_KEY`)

**Instalación:**
```bash
pip install anthropic mcp
```

**Código de ejemplo:**
```python
import anthropic
import asyncio
from mcp.client.sse import sse_client
from mcp import ClientSession

MCP_URL = "https://caad-138-100-11-141.ngrok-free.app/sse"

async def consultar_ehds(pregunta: str):
    async with sse_client(MCP_URL) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # Obtener herramientas disponibles
            tools = await session.list_tools()
            
            # Convertir al formato de Anthropic
            anthropic_tools = []
            for tool in tools.tools:
                anthropic_tools.append({
                    "name": tool.name,
                    "description": tool.description,
                    "input_schema": tool.inputSchema,
                })
            
            client = anthropic.Anthropic()
            messages = [{"role": "user", "content": pregunta}]
            
            # Bucle agéntico
            while True:
                response = client.messages.create(
                    model="claude-sonnet-4-20250514",
                    max_tokens=1024,
                    tools=anthropic_tools,
                    messages=messages,
                )
                
                if response.stop_reason == "end_turn":
                    for block in response.content:
                        if hasattr(block, "text"):
                            print(block.text)
                    break
                
                # Ejecutar llamadas a herramientas
                messages.append({"role": "assistant", "content": response.content})
                tool_results = []
                
                for block in response.content:
                    if block.type == "tool_use":
                        result = await session.call_tool(block.name, block.input)
                        result_text = result.content[0].text if result.content else "{}"
                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": result_text,
                        })
                
                messages.append({"role": "user", "content": tool_results})

# Ejecutar
asyncio.run(consultar_ehds("¿Qué conjuntos de datos hay disponibles?"))
```

---

## Opción 3: Ollama + mcphost (modelos locales, gratis)

**Requisitos:** 
- [Ollama](https://ollama.com) instalado
- [Go](https://golang.org) instalado
- mcphost instalado

**Instalación de mcphost:**
```bash
go install github.com/mark3labs/mcphost@latest
export PATH=$PATH:~/go/bin
```

**Descargar un modelo:**
```bash
ollama pull llama3.2
```

**Crear el archivo de configuración** (`~/.mcp.json`):
```json
{
  "mcpServers": {
    "ehds": {
      "url": "https://caad-138-100-11-141.ngrok-free.app/sse"
    }
  }
}
```

**Ejecutar:**
```bash
mcphost -m ollama:llama3.2
```

Se abrirá una sesión interactiva. Escribe tu pregunta y el modelo consultará automáticamente los datos del servidor.

**Ejemplo de sesión:**
```
> ¿Qué conjuntos de datos de salud están disponibles?

[Herramienta] ehds_list_datasets → 3 conjuntos encontrados

Hay 3 conjuntos de datos disponibles:
1. Cohorte sintética de Diabetes Tipo 2 — 48 pacientes
2. Cohorte sintética de Hipertensión Esencial — 50 pacientes  
3. Cohorte sintética de Síndrome Metabólico — 22 pacientes
```

---

## Opción 4: API de OpenAI (GPT-4o)

**Requisitos:** Clave API de OpenAI (`OPENAI_API_KEY`)

**Instalación:**
```bash
pip install openai mcp
```

**Código de ejemplo:**
```python
import openai
import asyncio
import json
from mcp.client.sse import sse_client
from mcp import ClientSession

MCP_URL = "https://caad-138-100-11-141.ngrok-free.app/sse"

async def consultar_ehds_gpt(pregunta: str):
    async with sse_client(MCP_URL) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            
            # Obtener herramientas disponibles
            tools_response = await session.list_tools()
            
            # Convertir al formato de OpenAI
            openai_tools = []
            for tool in tools_response.tools:
                openai_tools.append({
                    "type": "function",
                    "function": {
                        "name": tool.name,
                        "description": tool.description,
                        "parameters": tool.inputSchema,
                    }
                })
            
            client = openai.OpenAI()
            messages = [{"role": "user", "content": pregunta}]
            
            # Bucle agéntico
            while True:
                response = client.chat.completions.create(
                    model="gpt-4o",
                    messages=messages,
                    tools=openai_tools,
                    tool_choice="auto",
                )
                
                msg = response.choices[0].message
                messages.append(msg)
                
                if not msg.tool_calls:
                    print(msg.content)
                    break
                
                # Ejecutar llamadas a herramientas
                for tool_call in msg.tool_calls:
                    args = json.loads(tool_call.function.arguments)
                    result = await session.call_tool(tool_call.function.name, args)
                    result_text = result.content[0].text if result.content else "{}"
                    
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": result_text,
                    })

# Ejecutar
asyncio.run(consultar_ehds_gpt("¿Qué conjuntos de datos hay disponibles?"))
```

---

## Preguntas de ejemplo para probar

Estas preguntas funcionan bien con todas las opciones:

| Pregunta | Herramienta que se usa |
|----------|----------------------|
| "¿Qué conjuntos de datos de salud hay disponibles?" | `ehds_list_datasets` |
| "¿Cuál es el medicamento más común en el grupo de diabetes?" | `ehds_query_clinical` |
| "¿Cuántos pacientes femeninos hay en el grupo de hipertensión?" | `ehds_get_condition_stats` |
| "¿Puedo usar los datos de diabetes para entrenar un modelo comercial?" | `ehds_check_policy` |
| "¿Qué dataset tiene más pacientes?" | `ehds_list_datasets` |
| "Compara la distribución de género entre los tres grupos" | `ehds_get_condition_stats` (×3) |

---

## Herramientas disponibles en el servidor

| Herramienta | Descripción |
|-------------|-------------|
| `ehds_list_datasets` | Lista todos los conjuntos de datos del catálogo |
| `ehds_describe_dataset` | Metadatos completos de un conjunto de datos |
| `ehds_check_policy` | Política de uso ODRL (permisos y prohibiciones) |
| `ehds_search_datasets` | Buscar conjuntos de datos por palabra clave |
| `ehds_get_patients` | Lista de pacientes con datos demográficos |
| `ehds_get_condition_stats` | Estadísticas por código SNOMED CT |
| `ehds_query_clinical` | Consulta SPARQL libre sobre datos clínicos |

---

## Información técnica

- **Datos:** 120 pacientes sintéticos, 3,37 millones de triples RDF
- **Formato clínico:** FHIR R4 sobre RDF (HL7)
- **Catálogo:** HealthDCAT-AP Release 5 (estándar europeo EHDS)
- **Políticas:** ODRL 2.2 (uso solo para investigación, sin re-identificación)
- **Motor de consultas:** Apache Jena Fuseki 6 (SPARQL 1.1)
- **Protocolo IA:** MCP (Model Context Protocol), Anthropic, noviembre 2024
- **Licencia datos:** Creative Commons CC-BY 4.0

---

*Proyecto HARNESS — Horizon Europe grant 101169409*  
*Ontology Engineering Group, Universidad Politécnica de Madrid*
