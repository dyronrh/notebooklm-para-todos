# Guía Práctica de Prompt Engineering
## Técnicas para obtener mejores resultados con LLMs

---

## ¿QUÉ ES EL PROMPT ENGINEERING?

El prompt engineering es el arte y la ciencia de diseñar instrucciones efectivas para modelos de lenguaje (LLMs) como Claude, GPT-4 o Gemini. Un buen prompt puede marcar la diferencia entre una respuesta genérica y una solución precisa al problema.

**Principio fundamental**: Los LLMs son estadísticamente predecibles — producen la continuación más probable dado el prompt. Tu trabajo es diseñar el prompt de modo que la continuación más probable sea la que necesitas.

---

## TÉCNICA 1: ZERO-SHOT PROMPTING

Pedirle directamente al modelo sin ejemplos previos.

### Ejemplo básico (malo)
```
Analiza este contrato.
```

### Ejemplo mejorado
```
Analiza este contrato de arrendamiento e identifica:
1. Las partes involucradas
2. El monto de la renta mensual
3. Las condiciones de renovación
4. Las cláusulas que podrían representar un riesgo para el arrendatario
5. Cualquier término inusual o poco estándar

Contrato:
[PEGAR CONTRATO AQUÍ]
```

**Por qué funciona mejor**: Especificar el formato de la respuesta deseada y los aspectos concretos a analizar reduce la ambigüedad.

---

## TÉCNICA 2: FEW-SHOT PROMPTING

Proporcionar ejemplos de inputs y outputs esperados antes de tu pregunta real.

### Ejemplo: Clasificación de sentimientos

```
Clasifica el sentimiento del siguiente comentario de cliente como: Positivo, Neutro o Negativo.
Proporciona solo la clasificación sin explicación.

Comentario: "El producto llegó en perfectas condiciones y antes de lo esperado. ¡Muy recomendable!"
Clasificación: Positivo

Comentario: "El producto está bien, cumple con lo que describe. Sin más."
Clasificación: Neutro

Comentario: "Tardó el doble de lo prometido y el empaque llegó roto. Pésimo servicio."
Clasificación: Negativo

Comentario: "La calidad es buena pero el precio me pareció elevado para lo que ofrece."
Clasificación:
```

**Cuando usarlo**: Cuando necesitas respuestas en un formato muy específico o cuando el modelo necesita entender un patrón particular.

---

## TÉCNICA 3: CHAIN-OF-THOUGHT (COT)

Pedirle al modelo que razone paso a paso antes de dar la respuesta final. Aumenta significativamente la precisión en problemas complejos.

### Sin CoT (propenso a errores)
```
Si tengo 23 empleados y quiero formar equipos de 4, 
¿cuántos equipos completos puedo formar y cuántos empleados quedan sin equipo?
```

### Con CoT explícito
```
Si tengo 23 empleados y quiero formar equipos de 4, 
¿cuántos equipos completos puedo formar y cuántos empleados quedan sin equipo?

Piensa paso a paso antes de responder.
```

### Con CoT con estructura forzada
```
Problema: 23 empleados, equipos de 4 personas.

Resuelve siguiendo estos pasos:
1. Divide el total de empleados por el tamaño del equipo
2. El resultado entero (parte entera) es la cantidad de equipos completos
3. El resto es la cantidad de empleados sin equipo
4. Verifica tu resultado multiplicando y sumando

Muestra cada paso de tu cálculo.
```

### Cuándo usar CoT
- Problemas matemáticos
- Razonamiento multi-paso
- Análisis que requiere evaluar varios factores
- Decisiones que deben justificarse

---

## TÉCNICA 4: ROLE PROMPTING

Asignar un rol o persona al modelo para obtener respuestas desde esa perspectiva.

```
Eres un experto en marketing digital con 15 años de experiencia, especializado en 
empresas de e-commerce en América Latina. Hablas con claridad, usas ejemplos concretos 
y siempre basas tus recomendaciones en datos y casos reales.

[Tu pregunta aquí]
```

### Roles útiles por área

| Área | Rol recomendado |
|---|---|
| Código | "Eres un senior developer con 10 años en Python..." |
| Legal | "Eres un abogado especialista en derecho laboral argentino..." |
| Finanzas | "Eres un CFO con experiencia en startups de tecnología..." |
| Marketing | "Eres un consultor de branding especializado en PYMES..." |
| HR | "Eres una directora de RR.HH. de una empresa de 500 personas..." |

**Importante**: El role prompting funciona mejor cuando el rol incluye:
- Área de expertise
- Años de experiencia
- Especialización específica
- Estilo de comunicación esperado

---

## TÉCNICA 5: STRUCTURED OUTPUT PROMPTING

Forzar al modelo a responder en un formato estructurado específico.

### Solicitar respuesta en Markdown

```
Analiza el siguiente fragmento de código Python e identifica problemas.
Responde en el siguiente formato:

## Problemas encontrados

### Problema 1: [nombre del problema]
- **Tipo**: Bug / Performance / Security / Style
- **Línea(s)**: [número de línea]
- **Descripción**: [qué está mal]
- **Solución**: [cómo corregirlo]

### Problema 2: ...

## Resumen
- Total de problemas: [número]
- Severidad más alta: [crítico/medio/bajo]
- Prioridad de corrección: [lista ordenada]
```

### Solicitar respuesta en JSON

```
Extrae la siguiente información del texto y devuélvela como JSON válido:

Texto: [TEXTO AQUÍ]

Extrae:
- nombre_empresa (string)
- año_fundacion (number o null si no se menciona)
- empleados (number o null)
- pais (string)
- industria (string)
- facturacion_anual_usd (number o null)

Responde SOLO con el JSON, sin texto adicional.
```

---

## TÉCNICA 6: PROMPT CHAINING

Descomponer una tarea compleja en múltiples prompts secuenciales, donde el output de uno es el input del siguiente.

### Ejemplo: Análisis de propuesta de negocio

**Prompt 1** — Extracción:
```
Lee esta propuesta de negocio y extrae los siguientes datos en formato de lista:
- Problema que resuelve
- Solución propuesta
- Mercado objetivo
- Modelo de revenue
- Inversión solicitada
- Uso de los fondos
- Equipo fundador

[PROPUESTA]
```

**Prompt 2** — Análisis (usa el output del Prompt 1):
```
Basándote en los siguientes datos de una propuesta de negocio:

[OUTPUT DEL PROMPT 1]

Evalúa los siguientes aspectos en una escala del 1 al 10 con justificación:
1. Claridad del problema
2. Viabilidad de la solución
3. Tamaño del mercado
4. Modelo de negocio
5. Equipo
```

**Prompt 3** — Síntesis:
```
Basándote en el análisis anterior:

[OUTPUT DEL PROMPT 2]

Genera:
1. Un resumen ejecutivo de 100 palabras
2. Las 3 fortalezas principales
3. Las 3 debilidades principales
4. Una recomendación final: ¿Seguir adelante, modificar o descartar?
```

---

## TÉCNICA 7: CONSTRAINTS Y GUARDRAILS

Agregar restricciones explícitas para controlar el comportamiento del modelo.

```
Crea un plan de marketing para nuestra cafetería artesanal.

RESTRICCIONES:
- Presupuesto máximo: $2.000/mes
- Solo canales digitales (no impresos)
- Enfocado en Instagram y Google
- No incluir influencer marketing (no tenemos presupuesto)
- El tono debe ser cálido y auténtico, NO corporativo
- Las ideas deben ser implementables por una persona sin equipo
- EXCLUIR cualquier táctica que requiera más de 5 horas semanales de dedicación

FORMATO DE RESPUESTA:
- Máximo 5 tácticas
- Para cada táctica: nombre, descripción en 2 oraciones, costo estimado, horas semanales requeridas
```

---

## TÉCNICA 8: META-PROMPTING

Pedirle al modelo que mejore tu propio prompt.

```
El siguiente es un prompt que escribí para obtener un análisis de mi empresa:

"Analiza mi empresa de tecnología"

Por favor:
1. Identifica las ambigüedades y limitaciones de este prompt
2. Reescríbelo convirtiéndolo en un prompt más efectivo que obtenga respuestas más útiles y específicas
3. Explica los cambios que hiciste y por qué mejoran el resultado
```

---

## ERRORES COMUNES EN PROMPTING

### Error 1: Ambigüedad sin resolver
```
❌ "Hazlo más corto"
✅ "Resume este texto en máximo 3 oraciones, manteniendo los puntos más importantes"
```

### Error 2: Instrucciones negativas sin alternativa
```
❌ "No uses términos técnicos"
✅ "Usa lenguaje simple, como si le explicaras a alguien sin conocimientos del área. 
    Por ejemplo, en lugar de 'API', di 'sistema de comunicación entre programas'"
```

### Error 3: Demasiadas instrucciones conflictivas
```
❌ "Sé breve pero cubre todos los aspectos en detalle y da muchos ejemplos"
✅ "Resume en 5 bullets los aspectos más importantes. 
    Para cada bullet: 1 línea de explicación + 1 ejemplo concreto"
```

### Error 4: No especificar el formato
```
❌ "Dame ideas para mi negocio"
✅ "Dame 5 ideas de nuevos productos para una cafetería artesanal. 
    Para cada idea: nombre, descripción breve, precio sugerido, 
    dificultad de implementación (baja/media/alta)"
```

---

## PLANTILLAS LISTAS PARA USAR

### Para resumir documentos
```
Resume el siguiente [tipo de documento] siguiendo este formato:
- Propósito principal (1 oración)
- Puntos clave (máximo 5 bullets)
- Cifras/datos más importantes (si aplica)
- Conclusión o call to action (1 oración)
- Próximos pasos mencionados (si aplica)

Documento:
[DOCUMENTO]
```

### Para comparar opciones
```
Compara las siguientes [N] opciones para [objetivo]:

[OPCIONES]

Usa esta tabla:
| Criterio | Opción A | Opción B | Opción C |
| Costo | | | |
| [Criterio 2] | | | |
| [Criterio 3] | | | |

Después de la tabla, da tu recomendación en 2 oraciones justificando tu elección.
```

### Para generar ideas
```
Genera [N] ideas para [objetivo], teniendo en cuenta estas restricciones:
- [Restricción 1]
- [Restricción 2]

Para cada idea:
- Nombre: [nombre corto y descriptivo]
- Descripción: [2-3 oraciones]
- Ventaja principal: [por qué es buena idea]
- Desafío principal: [qué habría que resolver]
- Tiempo estimado de implementación: [días/semanas/meses]
```

---

*Sube este documento junto con el de Context Engineering y RAG a NotebookLM. Pregunta: "¿Cuál es la diferencia entre few-shot y zero-shot prompting?" o "Dame un ejemplo de chain-of-thought prompting para analizar contratos" o "¿Cuáles son los errores más comunes al escribir prompts?"*
