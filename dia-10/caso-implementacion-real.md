# Caso de Estudio: Implementación de NotebookLM en Empresa
## Constructora Andina S.A. — Gestión del Conocimiento Técnico

**Empresa**: Constructora Andina S.A.
**Industria**: Construcción e ingeniería civil
**País**: Perú
**Tamaño**: 1.200 empleados, 45 proyectos activos simultáneos
**Proyecto**: Digitalización de la gestión del conocimiento técnico
**Duración del piloto**: 3 meses (agosto-octubre 2024)
**Estado**: Piloto completado, en proceso de expansión corporativa

---

## CONTEXTO Y PROBLEMA

### La situación antes de NotebookLM

Constructora Andina S.A. es una empresa con 35 años de trayectoria en proyectos de infraestructura en Perú (carreteras, puentes, edificios residenciales y comerciales). Durante ese tiempo acumuló un volumen enorme de conocimiento técnico:

- **4.847 documentos técnicos** almacenados en SharePoint (especificaciones, planos técnicos en PDF, memorias descriptivas)
- **312 informes de proyecto** de obras completadas (lecciones aprendidas, informes finales, análisis de desviaciones)
- **89 manuales técnicos** de equipos y maquinaria
- **1.247 normativas y resoluciones** del MTC (Ministerio de Transportes) y otras entidades

**El problema central**: Este conocimiento era inaccesible en la práctica.

> "Teníamos toda la información, pero encontrarla tomaba horas. Un ingeniero que necesitaba saber qué dosificación de concreto se usó en el puente de Tarma en 2019 tenía que llamar a tres personas distintas o perderse durante dos horas en SharePoint. Muchos simplemente empezaban desde cero."
> — Ing. Carlos Palomino, Gerente de Ingeniería

**Consecuencias medibles** (antes del piloto):
- Tiempo promedio para encontrar un documento técnico: **47 minutos**
- Errores por repetición de estudios ya realizados: estimados en **S/. 380.000 al año**
- Proyectos que ignoraron aprendizajes de obras anteriores similares: **34% del total**
- Horas de ingeniero perdidas en búsqueda de información: **1.847 horas/mes** (equivalente a ~11 ingenieros a tiempo completo)

---

## DISEÑO DEL PILOTO

### Alcance

El piloto se limitó al área de Gerencia de Ingeniería con 3 casos de uso:

1. **Consulta técnica rápida**: "¿Qué especificación de asfalto usamos en la obra X?"
2. **Revisión de lecciones aprendidas**: "¿Qué problemas tuvimos en proyectos similares a este?"
3. **Preparación de licitaciones**: "¿Qué documentación base tenemos para la categoría de proyectos de movimiento de tierra?"

### Participantes

- 28 ingenieros del área de Ingeniería (civiles, estructurales, geotécnicos)
- 4 asistentes administrativos técnicos
- 2 gerentes de proyecto

### Documentos indexados en NotebookLM (piloto)

Se crearon **6 notebooks** temáticos:

| Notebook | Documentos | Páginas aprox. |
|---|---|---|
| Obras Viales — Especificaciones | 124 docs | 4.200 págs. |
| Obras Viales — Lecciones Aprendidas | 89 informes | 3.100 págs. |
| Obras de Edificación | 156 docs | 5.400 págs. |
| Normativa MTC Vigente | 78 normativas | 12.800 págs. |
| Maquinaria y Equipos | 45 manuales | 3.600 págs. |
| Estudios Geotécnicos — Base | 67 estudios | 2.900 págs. |

---

## IMPLEMENTACIÓN

### Semana 1-2: Preparación

**El mayor trabajo no fue técnico sino de organización documental.**

El equipo identificó que el 40% de los documentos tenían problemas:
- Nombres de archivo no descriptivos (ej: "Informe_v3_final_DEFINITIVO2.pdf")
- Documentos escaneados sin OCR (imagen de texto, no texto seleccionable)
- Versiones duplicadas o desactualizadas mezcladas con documentos vigentes

Acciones tomadas:
1. Contratar un servicio de OCR para 312 documentos escaneados
2. Renombrar y reorganizar archivos según taxonomía definida
3. Crear un Excel de registro: documento, versión vigente, sí/no indexar
4. Eliminar 847 documentos obsoletos del repositorio

**Lección clave**: La gestión documental previa es el 60% del trabajo de implementación.

### Semana 3: Configuración de NotebookLM

- Creación de los 6 notebooks
- Carga de documentos (proceso de 2 días por el volumen)
- Verificación de procesamiento correcto (test de 20 preguntas representativas)
- Capacitación inicial de 3 horas a todos los usuarios del piloto

### Semanas 4-12: Uso en producción

Los 32 participantes usaron NotebookLM en su trabajo diario durante 9 semanas.
Se registraron métricas semanalmente mediante formularios de Google.

---

## RESULTADOS DEL PILOTO

### Métricas de tiempo y eficiencia

| Métrica | Antes | Durante piloto | Mejora |
|---|---|---|---|
| Tiempo promedio búsqueda de documento | 47 min | 4.2 min | **-91%** |
| Horas/mes en búsqueda de información | 1.847 h | 198 h | **-89%** |
| % de proyectos que consultan lecciones aprendidas | 34% | 87% | **+53pp** |
| Preguntas respondidas sin necesidad de llamar a un colega | 23% | 78% | **+55pp** |

### Métricas de calidad

| Métrica | Resultado |
|---|---|
| Precisión de respuestas (verificación de muestra) | 94.3% |
| Tasa de "no encontrado" cuando la info sí existe | 3.8% |
| Satisfacción del usuario (NPS interno) | 72 |
| Usuarios que afirman que usarían la herramienta permanentemente | 91% |

### Impacto económico estimado (piloto)

| Concepto | Cálculo | Valor |
|---|---|---|
| Horas ahorradas (1.649 h × S/. 85/h promedio ingeniero) | 1.649 × 85 | S/. 140.165 |
| Reducción de estudios repetidos (estimado) | 2 estudios × S/. 25.000 | S/. 50.000 |
| Costo del piloto (horas de setup + costo herramienta) | — | S/. 38.400 |
| **ROI del piloto (3 meses)** | **(190.165 - 38.400) / 38.400** | **395%** |

---

## CASOS DE USO DESTACADOS

### Caso 1: La consulta geotécnica que tomó 3 minutos

**Situación**: Un ingeniero debía decidir el tipo de pilote para una fundación en Lima Norte. Necesitaba saber qué había hecho la empresa en terrenos similares.

**Antes**: Llamar al área de geotecnia, esperar respuesta, revisar archivos de 3 proyectos manualmente → 2-3 días.

**Con NotebookLM**: Consultó el notebook de Estudios Geotécnicos con: "¿Qué tipo de pilotes usamos en terrenos con capacidad portante menor a 1 kg/cm² en Lima?" → Respuesta en 45 segundos, con citas exactas de 4 estudios anteriores relevantes.

### Caso 2: Preparación de propuesta técnica en 2 horas

**Situación**: El equipo comercial necesitaba preparar la parte técnica de una licitación de carretera en Cusco en tiempo récord (24 horas).

**Con NotebookLM**: Preguntaron a los notebooks de Obras Viales: "Genera un resumen de nuestra experiencia en construcción de carreteras en sierra con altitudes sobre 3.000 msnm." En 2 horas, tenían un borrador estructurado que normalmente habría tomado 2 días.

### Caso 3: Normativa al día

**Situación**: Cambio de normativa del MTC a mediados del piloto. El equipo necesitaba entender el impacto en proyectos activos.

**Con NotebookLM**: Subieron la nueva normativa y preguntaron: "¿Qué requisitos de esta nueva normativa son diferentes a los anteriores y podrían afectar nuestros proyectos activos de pavimentación?" Respuesta citada y estructurada en minutos.

---

## DESAFÍOS Y LIMITACIONES ENCONTRADAS

### Desafío 1: Resistencia cultural
**Problema**: 8 de los 32 usuarios (25%) mostraron resistencia a cambiar sus hábitos. "Prefiero llamar a Carlos que depender de una máquina."
**Solución**: Sesiones de demostración 1:1 con casos de uso específicos de cada área. En 4 semanas, todos usaban la herramienta.

### Desafío 2: Documentos escaneados (PDFs imagen)
**Problema**: 40% de los documentos históricos eran imágenes, no texto seleccionable. NotebookLM no puede leer texto en imágenes.
**Solución**: Inversión en OCR para los documentos más críticos. Los menos relevantes se dejaron sin indexar.

### Desafío 3: Documentos en mal estado
**Problema**: Muchos documentos tenían tablas complejas, planos CAD embebidos o formatos no estándar.
**Solución**: Para planos y tablas complejas, se generaron resúmenes en texto plano que sí se indexaron.

### Desafío 4: Límite de fuentes por notebook
**Problema**: NotebookLM tiene un límite de 50 fuentes por notebook. Para algunas categorías tenían más documentos.
**Solución**: Crear múltiples notebooks temáticos y capacitar al equipo para saber cuál consultar según el caso.

### Desafío 5: Información desactualizada en los documentos
**Problema**: Algunos documentos tenían especificaciones antiguas. NotebookLM las presentaba con igual confianza que las vigentes.
**Solución**: Agregar metadatos claros de fecha y versión al inicio de cada documento, y establecer un proceso de revisión anual para actualizar o archivar documentos.

---

## PLAN DE EXPANSIÓN 2025

Con el éxito del piloto, el Directorio aprobó la expansión:

**Q1 2025**: Expandir a todas las áreas de la empresa (total 1.200 empleados)
**Q2 2025**: Implementar NotebookLM Plus para equipos con alta demanda
**Q3 2025**: Crear un proceso formal de "gestión del conocimiento" con responsables por área
**Q4 2025**: Auditoría anual y planificación 2026

**Inversión aprobada**: S/. 245.000 (incluye digitalización documental, capacitación y herramienta)
**ROI proyectado a 12 meses**: 820%

---

*Este caso de estudio fue preparado como material de referencia para la implementación en otras empresas. Sube este documento junto con el análisis de ROI y las mejores prácticas a NotebookLM y pregunta: "¿Cuáles fueron los principales desafíos de la implementación?" o "¿Qué resultados obtuvo la empresa en el piloto de 3 meses?" o "¿Qué casos de uso tuvieron mayor impacto?"*
