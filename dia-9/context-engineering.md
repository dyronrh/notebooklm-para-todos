# Context Engineering: El Arte de Preparar Información para la IA
## Guía Avanzada para Maximizar el Rendimiento de los LLMs

---

## ¿POR QUÉ EL CONTEXTO LO ES TODO?

Los modelos de lenguaje son, en esencia, máquinas de completar contexto. Dada una secuencia de texto (el contexto), predicen la continuación más probable. El **context engineering** es la disciplina de diseñar ese contexto para que la continuación más probable sea exactamente lo que necesitas.

Un modelo mediocre con un contexto excelente supera a un modelo excelente con un contexto pobre.

---

## LA ANATOMÍA DEL CONTEXTO PERFECTO

```
┌─────────────────────────────────────────────────────────┐
│                    VENTANA DE CONTEXTO                   │
├─────────────────────────────────────────────────────────┤
│  1. SYSTEM PROMPT                                        │
│     Rol + instrucciones + restricciones + formato        │
├─────────────────────────────────────────────────────────┤
│  2. CONOCIMIENTO INYECTADO                               │
│     Documentos relevantes + datos actualizados           │
│     (aquí es donde NotebookLM y RAG aportan valor)       │
├─────────────────────────────────────────────────────────┤
│  3. EJEMPLOS (FEW-SHOT)                                  │
│     Input → Output ejemplares para calibrar el modelo    │
├─────────────────────────────────────────────────────────┤
│  4. HISTORIAL DE CONVERSACIÓN                            │
│     Intercambios previos relevantes                      │
├─────────────────────────────────────────────────────────┤
│  5. QUERY DEL USUARIO                                    │
│     La pregunta o tarea actual                           │
└─────────────────────────────────────────────────────────┘
```

---

## ELEMENTO 1: EL SYSTEM PROMPT

El system prompt define el comportamiento global del sistema. Es la configuración de fábrica del asistente.

### Estructura de un system prompt efectivo

```
[ROL Y EXPERTISE]
Eres [rol] con experiencia en [dominio específico].
[Características adicionales del personaje si son relevantes]

[OBJETIVO PRINCIPAL]
Tu función es [qué hace el sistema].

[INSTRUCCIONES CLAVE]
- [Instrucción 1]
- [Instrucción 2]
- [Instrucción 3]

[RESTRICCIONES]
- [Lo que NO debes hacer]
- [Limitaciones de alcance]

[FORMATO DE RESPUESTA]
Responde siempre en [idioma/formato/estructura].
```

### Ejemplo: Asistente de soporte técnico

```
Eres el asistente técnico de TechSoluciones Empresariales, especializado en 
hardware y redes para empresas.

Tu función es ayudar a clientes con dudas sobre productos, especificaciones técnicas, 
compatibilidad y problemas comunes.

Instrucciones:
- Responde SIEMPRE en español
- Basa tus respuestas únicamente en la documentación técnica proporcionada
- Si no sabes la respuesta, indica claramente "No tengo esa información" y sugiere 
  contactar soporte@techsoluciones.com.ar
- Sé preciso con las especificaciones técnicas (no redondees cifras)
- Si hay varias opciones para un problema, preséntalas en orden de complejidad (menor a mayor)

Restricciones:
- NO hagas comparaciones con precios de competidores
- NO hagas promesas de tiempo de entrega sin consultar stock
- NO accedas a información que no esté en los documentos técnicos

Formato: 
- Respuestas cortas (< 200 palabras) para preguntas simples
- Para problemas técnicos complejos: usa pasos numerados
- Siempre termina con: "¿Hay algo más en lo que pueda ayudarte?"
```

---

## ELEMENTO 2: CONOCIMIENTO INYECTADO

Esta es la parte más crítica del context engineering y donde NotebookLM brilla.

### Estrategias de selección de información

**Relevancia semántica** (lo que hace NotebookLM/RAG):
Recuperar los fragmentos de texto que más se parecen semánticamente a la pregunta.

**Relevancia temporal** (para datos que cambian):
Priorizar la información más reciente cuando hay versiones múltiples.

**Relevancia jerárquica**:
Incluir tanto el detalle específico como el contexto general.

```
Contexto malo:
"La garantía es de 12 meses."

Contexto mejor:
"[Política de garantías — TechSoluciones — Vigente enero 2024]
Todos los productos de la línea corporativa tienen garantía de 12 meses 
desde la fecha de factura. La garantía cubre defectos de fabricación.
NO cubre: daños físicos, líquidos, mal uso, o modificaciones no autorizadas.
Para reclamar garantía: soporte@techsoluciones.com.ar o 0800-XXX-XXXX."
```

El segundo ejemplo es mejor porque incluye:
- Metadatos (quién, cuándo)
- Contexto de aplicación (línea corporativa)
- Excepciones importantes
- Siguiente paso de acción

### El problema de la "aguja en el pajar"

Si inyectas demasiado contexto irrelevante, el modelo puede:
- Perder el foco en lo que realmente importa
- Mezclar información de diferentes fuentes incorrectamente
- Aumentar la latencia y el costo

**Regla práctica**: Inyecta suficiente contexto para responder la pregunta, pero no más.

**Para NotebookLM**: Esto lo resuelve automáticamente. Solo sube los documentos relevantes para el caso de uso.

### Enriquecimiento del contexto

Puedes mejorar la información antes de inyectarla:

```python
# Contexto básico (crudo)
contexto_crudo = """
Facturación Q3: 847M
Empleados: 4847
Margen: 24%
"""

# Contexto enriquecido (con estructura y metadatos)
contexto_enriquecido = """
=== INFORME FINANCIERO Q3 2024 — GRUPO MERIDIAN ===
Fecha: 30/09/2024 | Fuente: Estado de resultados auditado

MÉTRICAS PRINCIPALES:
- Facturación total: $847 millones USD (+18.3% vs Q3 2023)
- EBITDA margin: 24.0% (mejora de +2.1pp vs año anterior)
- Total colaboradores: 4.847 (+23.9% vs año anterior)

CONTEXTO: Este trimestre superó en +4.2% las proyecciones del plan estratégico.
===
"""
```

---

## ELEMENTO 3: COMPRESIÓN DE CONTEXTO

Cuando tienes más información de la que cabe en la ventana de contexto, necesitas comprimir.

### Técnicas de compresión

**Resumen jerárquico**:
```
Documento original: 50 páginas
→ Resumen detallado: 5 páginas (por sección)
→ Resumen ejecutivo: 1 página
→ Bullets clave: 10 puntos
```

**Filtrado por relevancia**:
Solo incluir las secciones relacionadas con la consulta actual.

**Extracción de datos estructurados**:
Convertir prosa en tablas o listas cuando la información es estructurada.

```
ANTES (prosa, 150 palabras):
"La empresa fue fundada en 1996 por Roberto Alarcón... 
tiene 287 empleados distribuidos en Valparaíso, Santiago y Rancagua...
su facturación anual ronda los $18.500 millones..."

DESPUÉS (datos, 30 palabras):
Empresa: DNP | Fundación: 1996 | Fundador: R. Alarcón
Empleados: 287 | Regiones: 3 (V, RM, VI)  
Facturación: $18.500M CLP/año
```

---

## ELEMENT 4: CONVERSACIÓN Y MEMORIA

### Tipos de memoria en sistemas de IA

**Memoria en contexto** (corto plazo):
El historial de la conversación actual. Limitado por la ventana de contexto.

**Memoria externa** (largo plazo):
Base de datos donde se almacenan hechos importantes de conversaciones anteriores.

```python
# Sistema de memoria simple
class AgentMemory:
    def __init__(self):
        self.short_term = []  # Conversación actual
        self.long_term = {}   # Hechos persistentes sobre el usuario
    
    def add_message(self, role: str, content: str):
        self.short_term.append({"role": role, "content": content})
    
    def remember_fact(self, key: str, value: str):
        # Ej: "usuario_empresa" → "TechStartup SA"
        self.long_term[key] = value
    
    def get_context_for_prompt(self) -> str:
        facts = "\n".join([f"- {k}: {v}" for k, v in self.long_term.items()])
        return f"""
Información del usuario:
{facts}

Conversación reciente:
[últimos N turnos del short_term]
"""
```

### Gestión del historial en NotebookLM

NotebookLM mantiene el historial de la conversación dentro de cada sesión de chat. Para una sesión nueva, el historial se reinicia (excepto las notas guardadas). Para proyectos de largo aliento:
- Guarda las respuestas importantes como **notas** (botón "Save to note")
- Las notas persisten entre sesiones y pueden usarse como fuentes adicionales
- Puedes pedir a NotebookLM que consolide varias notas en un documento maestro

---

## APLICACIÓN PRÁCTICA: DISEÑAR UN SISTEMA CON NOTEBOOKLM

### Caso: Base de conocimiento para servicio al cliente

**Objetivo**: Que los agentes de soporte puedan responder preguntas sobre políticas y productos sin memorizar documentos.

**Paso 1: Preparar los documentos**
```
✅ Incluir:
- Manual del producto (bien estructurado con headers)
- Política de garantías y devoluciones
- FAQ actualizada
- Lista de precios vigente
- Procedimientos de soporte técnico

❌ No incluir:
- Borradores o versiones antiguas
- Documentos internos no relacionados
- Documentación de procesos que el agente no necesita
```

**Paso 2: Crear el notebook**
- Un notebook por línea de productos o área de soporte
- Nombre descriptivo: "Soporte — Línea Laptops Corporativas Q4 2024"

**Paso 3: Definir preguntas tipo**
```
Preguntas que los agentes deben poder responder:
- ¿Cuál es la garantía del producto X?
- ¿Qué cubre la garantía y qué no?
- ¿Cuánto demora el proceso de devolución?
- ¿Cuáles son los requisitos para solicitar un reemplazo?
- ¿Cuál es el proceso de soporte técnico?
```

**Paso 4: Probar con preguntas difíciles**
```
Prueba estas preguntas edge case:
- "¿Qué pasa si el producto se rompe después de la garantía?"
- "¿Puedo reclamar garantía si lo compré en cuotas y aún debo?"
- "¿La garantía aplica si lo usa otro empleado de la empresa?"
```

**Paso 5: Iterar**
Si NotebookLM no puede responder alguna pregunta válida → agregar esa información a los documentos.

---

## MÉTRICAS PARA EVALUAR LA CALIDAD DEL CONTEXTO

| Métrica | Pregunta | Cómo medirla |
|---|---|---|
| **Completitud** | ¿El contexto tiene toda la info necesaria? | ¿El modelo responde "no tengo esa información" cuando sí debería tenerla? |
| **Precisión** | ¿La información es correcta? | Verificar respuestas contra fuentes originales |
| **Relevancia** | ¿El contexto es pertinente a las preguntas? | % de contexto recuperado que realmente se usó en la respuesta |
| **Consistencia** | ¿El contexto no tiene contradicciones? | ¿El modelo da respuestas contradictorias según qué fragmento recupera? |
| **Actualidad** | ¿La información está vigente? | Verificar fechas de los documentos fuente |

---

*Este documento es el más técnico de la serie. Súbelo junto con los documentos de RAG, Prompt Engineering y Arquitectura de Agentes para tener una base de conocimiento completa sobre IA aplicada. Pregunta a NotebookLM: "¿Cuáles son los 5 elementos de la ventana de contexto?" o "¿Cómo debería preparar mis documentos para que un agente de IA los use efectivamente?"*
