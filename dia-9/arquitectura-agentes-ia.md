# Arquitectura de Agentes de IA
## Guía para Construir Sistemas Agénticos con LLMs

---

## ¿QUÉ ES UN AGENTE DE IA?

Un **agente de IA** es un sistema que usa un LLM como "cerebro" para percibir el entorno, razonar y tomar acciones para alcanzar un objetivo.

A diferencia de un LLM convencional (que solo responde a preguntas), un agente puede:
- **Usar herramientas**: buscar en Google, ejecutar código, llamar APIs, consultar bases de datos
- **Tomar múltiples pasos**: planificar y ejecutar secuencias de acciones
- **Adaptarse**: observar los resultados de sus acciones y ajustar el plan
- **Mantener estado**: recordar lo que ha hecho en conversaciones largas

---

## ANATOMÍA DE UN AGENTE

```
┌─────────────────────────────────────────────┐
│                  AGENTE                      │
│                                              │
│  ┌─────────┐    ┌─────────┐   ┌──────────┐  │
│  │Percepción│→  │Razona-  │→  │  Acción  │  │
│  │(Input)  │   │ miento   │   │ (Output) │  │
│  │         │   │ (LLM)   │   │          │  │
│  └─────────┘   └─────────┘   └────┬─────┘  │
│       ↑                           │         │
│       └──────── Observación ───────┘         │
│                 (Feedback)                   │
└─────────────────────────────────────────────┘
         ↕ Herramientas (Tools)
    [Búsqueda] [Código] [APIs] [DB] [Email]
```

---

## PATRONES DE ARQUITECTURA AGÉNTICA

### Patrón 1: ReAct (Reasoning + Acting)

El patrón más común. El LLM alterna entre razonamiento y acción en un loop.

```
Thought: Necesito saber el precio del dólar hoy para calcular la conversión.
Action: buscar_web("precio dólar hoy Argentina")
Observation: El tipo de cambio oficial es $1.050 ARS/USD (fecha: 15/11/2024)

Thought: Ahora puedo calcular. El usuario tiene $5.000 USD.
Action: calcular("5000 * 1050")
Observation: Resultado = 5.250.000

Thought: Tengo el resultado. Puedo responder al usuario.
Final Answer: $5.000 USD equivalen a $5.250.000 ARS al tipo de cambio oficial actual.
```

### Patrón 2: Plan-and-Execute

El agente primero genera un plan completo y luego lo ejecuta paso a paso.

```
Objetivo: "Analiza las ventas del Q3 y genera un informe ejecutivo"

FASE DE PLANIFICACIÓN:
1. Obtener datos de ventas de Q3 de la base de datos
2. Calcular totales por región, producto y vendedor
3. Comparar con Q3 del año anterior
4. Identificar los 3 insights más relevantes
5. Generar el informe en formato Markdown

FASE DE EJECUCIÓN:
[Ejecuta cada paso del plan usando las herramientas disponibles]
```

### Patrón 3: Multi-Agent (Sistemas Multiagente)

Múltiples agentes especializados colaboran para resolver problemas complejos.

```
                    ┌──────────────────┐
                    │  Agente Orquesta │
                    │  (Coordinador)   │
                    └───────┬──────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼──────┐   ┌────────▼──────┐   ┌────────▼──────┐
│   Agente     │   │    Agente     │   │    Agente     │
│  Investigador │   │  Analista    │   │  Redactor     │
│              │   │              │   │              │
│ [Web search] │   │ [Python code]│   │ [Sin tools]  │
│ [arxiv API]  │   │ [Excel read] │   │              │
└──────────────┘   └─────────────-┘   └──────────────┘
```

**Ejemplo de uso**: Un sistema de análisis de mercado donde:
- El Agente Investigador busca información sobre competidores
- El Agente Analista procesa los datos y genera estadísticas
- El Agente Redactor escribe el informe final

---

## HERRAMIENTAS (TOOLS / FUNCTION CALLING)

Las herramientas son funciones que el LLM puede llamar para interactuar con el mundo.

### Definición de una herramienta (formato Claude API)

```python
tools = [
    {
        "name": "search_web",
        "description": "Busca información actualizada en internet. Úsala cuando necesites datos recientes o que no están en tu conocimiento de entrenamiento.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "La consulta de búsqueda en lenguaje natural"
                },
                "num_results": {
                    "type": "integer",
                    "description": "Número de resultados a retornar (1-10)",
                    "default": 5
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "execute_python",
        "description": "Ejecuta código Python y retorna el resultado. Úsala para cálculos, análisis de datos, o procesamiento.",
        "input_schema": {
            "type": "object",
            "properties": {
                "code": {
                    "type": "string",
                    "description": "El código Python a ejecutar"
                }
            },
            "required": ["code"]
        }
    },
    {
        "name": "read_database",
        "description": "Ejecuta una consulta SQL en la base de datos de la empresa y retorna los resultados.",
        "input_schema": {
            "type": "object",
            "properties": {
                "sql_query": {
                    "type": "string",
                    "description": "La consulta SQL a ejecutar (solo SELECT, no modificaciones)"
                }
            },
            "required": ["sql_query"]
        }
    }
]
```

### Loop de ejecución de herramientas (Python)

```python
import anthropic
import json

client = anthropic.Anthropic()

def run_agent(user_message: str, tools: list, tool_functions: dict) -> str:
    messages = [{"role": "user", "content": user_message}]
    
    while True:
        # Llamar al LLM
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )
        
        # Si el modelo terminó (no quiere usar más herramientas)
        if response.stop_reason == "end_turn":
            # Extraer el texto de respuesta final
            return next(
                block.text 
                for block in response.content 
                if block.type == "text"
            )
        
        # Si el modelo quiere usar herramientas
        if response.stop_reason == "tool_use":
            # Agregar la respuesta del asistente al historial
            messages.append({"role": "assistant", "content": response.content})
            
            # Ejecutar cada herramienta solicitada
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    tool_name = block.name
                    tool_input = block.input
                    
                    print(f"🔧 Ejecutando herramienta: {tool_name}")
                    print(f"   Input: {json.dumps(tool_input, indent=2)}")
                    
                    # Ejecutar la función real
                    if tool_name in tool_functions:
                        result = tool_functions[tool_name](**tool_input)
                    else:
                        result = f"Error: herramienta '{tool_name}' no encontrada"
                    
                    print(f"   Output: {str(result)[:200]}")
                    
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })
            
            # Agregar los resultados al historial
            messages.append({"role": "user", "content": tool_results})

# Ejemplo de uso
respuesta = run_agent(
    user_message="¿Cuánto valen las acciones de Apple hoy y cuánto es eso en pesos argentinos?",
    tools=tools,
    tool_functions={
        "search_web": search_web_function,
        "execute_python": execute_python_function,
    }
)
```

---

## NOTEBOOKLM COMO AGENTE DE CONOCIMIENTO

NotebookLM puede pensarse como un **agente de conocimiento especializado** con las siguientes características:

**Herramientas implícitas**:
- `search_in_sources(query)`: Busca semánticamente en tus documentos
- `generate_audio_overview()`: Genera el podcast de resumen
- `create_study_guide()`: Genera una guía de estudio estructurada
- `generate_quiz()`: Crea preguntas de evaluación

**Sistema de razonamiento**: El modelo de Gemini de Google razona sobre el contenido de tus fuentes antes de responder.

**Restricción importante**: NotebookLM solo puede "actuar" dentro del contexto de tus documentos. No busca en internet, no ejecuta código, no llama APIs externas. Esto es una limitación intencionada que garantiza respuestas confiables y citadas.

**Cuándo NotebookLM es el agente correcto**:
- Consultas de conocimiento basadas en documentos específicos
- Síntesis y análisis de información existente
- Educación y preparación basada en materiales definidos

**Cuándo necesitas un agente más capaz**:
- Necesitas datos en tiempo real (precios, noticias, métricas)
- Necesitas ejecutar acciones (enviar emails, modificar bases de datos)
- Necesitas integrar múltiples sistemas
- El flujo de trabajo requiere múltiples pasos complejos con decisiones

---

## PREPARAR INFORMACIÓN PARA AGENTES DE IA

Si estás construyendo un agente que trabajará con documentos de tu empresa, la preparación de esos documentos es crítica. Los mismos principios que aplican para NotebookLM aplican para sistemas RAG que alimentan agentes.

### Guía de preparación de documentos para agentes

1. **Normaliza el formato**: Convierte todo a texto plano o Markdown estructurado
2. **Agrega metadatos en el documento**: Fecha, versión, autor, área
3. **Usa headers descriptivos**: El agente los usará como señales de navegación
4. **Sé explícito con los términos**: Define acrónimos y términos internos en el mismo documento
5. **Separa el "qué" del "cómo"**: Los procedimientos operativos deben tener pasos claros y enumerados
6. **Agrega ejemplos**: Los agentes (como los humanos) aprenden mejor con ejemplos concretos

---

*Sube este documento junto con los de RAG y Prompt Engineering a NotebookLM y pregunta: "¿Cuál es la diferencia entre el patrón ReAct y Plan-and-Execute?" o "¿Qué es function calling y para qué sirve?" o "¿Por qué NotebookLM tiene restricciones en las acciones que puede tomar?"*
