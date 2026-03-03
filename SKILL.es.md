---
name: auto-watcher
version: 1.2.0
description: "Observador automático de archivos y tareas impulsado por IA para automatización de investigación"
author: OpenClaw Team
tags: [research, ai, automation]
dependencies:
  - python>=3.8
  - watchdog>=3.0.0
  - openai>=4.0.0
  - pydantic>=2.0.0
  - redis>=4.5.0
  - structlog>=23.0.0
required_env_vars:
  - AUTO_WATCHER_OPENAI_API_KEY
  - AUTO_WATCHER_REDIS_URL
optional_env_vars:
  - AUTO_WATCHER_LOG_LEVEL=INFO
  - AUTO_WATCHER_POLL_INTERVAL=1.0
  - AUTO_WATCHER_MAX_BATCH_SIZE=10
  - AUTO_WATCHER_AI_MODEL=gpt-4
platforms: [linux, darwin]
---

# Observador Automático

Observador automático de archivos y tareas impulsado por IA para pipelines de automatización de investigación.

## Propósito

Observador Automático monitorea directorios y colas de tareas, activando automáticamente análisis con IA cuando los archivos cambian o llegan nuevos datos de investigación. Elimina la conexión manual entre la ingesta de datos y el procesamiento con IA en flujos de trabajo de investigación.

**Casos de uso reales:**
- Monitorear `/data/papers/` para nuevos PDFs → extraer texto automáticamente → resumir
- Monitorear `/data/experiments/` para nuevos resultados CSV → analizar tendencias automáticamente → generar informe
- Monitorear `/data/notes/` para cambios en markdown → sugerir automáticamente papers relacionados mediante búsqueda semántica
- Monitorear repositorios Git para nuevos commits → generar automáticamente logs de investigación y resúmenes
- Monitorear `/data/raw/` para nuevos conjuntos de datos → validar esquema automáticamente → activar pipelines descendentes

## Alcance

**Comandos:**
- `autowatch start <config_file>` - Inicia el observador con configuración YAML
- `autowatch stop` - Detiene las instancias del observador en ejecución
- `autowatch status` - Muestra los observadores activos y estadísticas de la cola
- `autowatch config validate <file>` - Valida el esquema de configuración
- `autowatch config generate <template>` - Genera un esqueleto de configuración
- `autowatch queue list` - Lista las tareas pendientes y fallidas
- `autowatch queue retry <task_id>` - Reintenta la tarea fallida
- `autowatch metrics` - Muestra métricas de throughput y latencia

**Estructura de archivo de configuración:**
```yaml
watchers:
  - path: /data/papers/incoming
    patterns: ["*.pdf", "*.md"]
    ignore_patterns: ["*.tmp"]
    event_types: [created, modified]
    ai_pipeline: paper-summarizer
    batch_size: 5
    debounce_ms: 2000
  - path: /data/experiments/results
    patterns: ["*.csv"]
    event_types: [created]
    ai_pipeline: experiment-analyzer
    condition: "file.size > 1024"
ai_pipelines:
  paper-summarizer:
    model: "gpt-4"
    prompt_file: ./prompts/summarize.md
    max_tokens: 500
    output_format: json
  experiment-analyzer:
    model: "gpt-4-turbo"
    prompt_template: "Analyze: {{file_path}}"
    context_window: 8192
queue:
  backend: redis
  max_retries: 3
  retry_delay_s: 60
storage:
  results_backend: redis
  archive_path: /data/processed/
```

## Proceso de Trabajo

1. **Inicialización**
   ```bash
   autowatch config validate ./research-watcher.yaml
   autowatch start ./research-watcher.yaml
   ```

2. **Detección de Eventos**
   - Usa `watchdog` para monitorear eventos del sistema de archivos
   - Los eventos tienen un debounce según la configuración del observador (`debounce_ms`)
   - Se recopilan metadatos del archivo: ruta, tamaño, mtime, tipo de evento

3. **Filtrado**
   - Aplica filtros glob `patterns` y `ignore_patterns`
   - Evalúa expresiones `condition` (Python `eval` con espacio de nombres seguro)
   - Los archivos que no pasan los filtros se registran y descartan

4. **Agrupación**
   - Acumula eventos hasta `batch_size` o tiempo de espera `debounce_ms`
   - Entrega el lote a la pipeline de IA como una única solicitud

5. **Procesamiento con IA**
   - Carga la configuración de la pipeline (`ai_pipelines.<name>`)
   - Renderiza la plantilla de prompt con contexto (`file_path`, `file_content`, metadatos)
   - Llama a la API de OpenAI con el modelo y parámetros especificados
   - Parsea `output_format` (json/text/markdown)

6. **Manejo de Resultados**
   - Almacena el resultado en Redis con clave: `autowatch:result:<hash(file_path)>`
   - Si se establece `archive_path`, mueve el archivo procesado al archivo
   - Publica evento: `autowatch.completed` con metadatos del resultado

7. **Manejo de Errores**
   - Reintenta hasta `max_retries` en fallos transitorios
   - Después de agotar los reintentos, marca la tarea como `failed` y publica `autowatch.failed`
   - Las tareas fallidas pueden ser reencoladas manualmente con `autowatch queue retry`

8. **Métricas**
   - Métricas de Prometheus expuestas en `:9090/metrics` si `metrics_enabled: true`
   - Rastrea: `events_received`, `batches_processed`, `ai_calls_total`, `processing_duration_seconds`, `errors_total`

## Reglas de Oro

- Nunca ejecutar Auto Watcher como root; usar el usuario dedicado `autowatch`
- Siempre validar la configuración antes de iniciar: `autowatch config validate`
- Establecer `AUTO_WATCHER_OPENAI_API_KEY` en archivo `.env`, nunca en el YAML de configuración
- Asegurar que `archive_path` esté en el mismo sistema de archivos para evitar errores de enlace entre dispositivos
- Para producción, ejecutar con unidad `systemd` y `Restart=on-failure`
- No apuntar observadores a directorios con >10k archivos; usar subdirectorios específicos
- Siempre establecer `batch_size` apropiado a los límites de tasa de tu modelo de IA
- Monitorear eventos `autowatch.failed`; investigar tasa de fallos >1%
- Usar base de datos Redis separada (`db=1`) de otras aplicaciones
- Mantener prompts de pipeline de IA en control de versiones, no incrustados en la configuración

## Ejemplos

**Ejemplo 1: Monitorear directorio de papers**
```bash
# config: papers-watcher.yaml
autowatch config validate papers-watcher.yaml
autowatch start papers-watcher.yaml
```

Archivo de entrada: `/data/papers/incoming/attention-is-all-you-need.pdf`
Evento: Created
Prompt de IA renderizado:
```
Summarize the following research paper, extracting: 1) Main contribution, 2) Methodology, 3) Results, 4) Limitations.

Paper: [Extracted text from PDF...]
```
Salida (almacenada en Redis):
```json
{
  "file": "/data/papers/incoming/attention-is-all-you-need.pdf",
  "summary": "Introduces Transformer architecture...",
  "key_points": ["Self-attention", "No recurrence", "State-of-the-art on MT"],
  "timestamp": "2026-03-03T10:23:45Z"
}
```

**Ejemplo 2: Monitorear resultados experimentales**
```bash
autowatch queue list
# Output:
# ID    Status    Created              Pipeline
# a1b2c pending  2026-03-03T10:25:00Z experiment-analyzer
# d4e5f failed   2026-03-03T10:24:15Z experiment-analyzer

autowatch queue retry d4e5f
# Task requeued, new ID: x9y8z
```

**Ejemplo 3: Verificación de estado**
```bash
autowatch status
# Output:
# Watcher: papers-watcher (running)
#   Active watchers: 3
#   Events processed: 1,247
#   Batches processed: 142
#   Queue depth: 5 (redis)
#   AI calls: 142 (avg duration: 2.3s)
#   Errors: 2 (retrying)
```

## Comandos de Reversión

**Detener observador:**
```bash
autowatch stop
# Verifies: no watcher processes remain (`pgrep -f autowatch`)
```

**Restaurar archivos archivados:**
```bash
# If archive_path is /data/processed/
find /data/processed/ -name "*.processed" -exec sh -c '
  mv "$1" "$(echo "$1" | sed "s|/data/processed/|/data/papers/incoming/|" | sed "s|\.processed$||")"
' _ {} \;
```

**Limpiar cola Redis (peligroso - usar con precaución):**
```bash
redis-cli -n 1 DEL autowatch:queue:*
redis-cli -n 1 DEL autowatch:results:*
```

**Revertir cambios de configuración:**
```bash
# Assuming config in git
git checkout HEAD~1 -- ./research-watcher.yaml
# Validate and restart
autowatch config validate ./research-watcher.yaml && autowatch restart ./research-watcher.yaml
```

**Recuperarse de Redis corrupto:**
```bash
# Flush specific DB and restart watcher
redis-cli -n 1 FLUSHDB
autowatch stop && autowatch start ./research-watcher.yaml
# Reprocess archived files manually if needed
```

**Deshabilitar un observador específico sin detener el sistema completo:**
```bash
# Edit config, set enabled: false for that watcher
sed -i 's/enabled: true/enabled: false/' ./research-watcher.yaml
autowatch config validate ./research-watcher.yaml && autowatch reload ./research-watcher.yaml
```
```