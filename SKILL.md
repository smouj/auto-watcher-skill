---
name: auto-watcher
version: 1.2.0
description: AI-powered automatic file and task watcher for research automation
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

# Auto Watcher

AI-powered automatic file and task watcher for research automation pipelines.

## Purpose

Auto Watcher monitors directories and task queues, automatically triggering AI analysis when files change or new research data arrives. It eliminates manual gluing between data ingestion and AI processing in research workflows.

**Real use cases:**
- Watch `/data/papers/` for new PDFs → auto-extract text → summarize
- Monitor `/data/experiments/` for new CSV results → auto-analyze trends → generate report
- Watch `/data/notes/` for markdown changes → auto-suggest related papers via semantic search
- Monitor Git repos for new commits → auto-generate research logs and summaries
- Watch `/data/raw/` for new datasets → auto-validate schema → trigger downstream pipelines

## Scope

**Commands:**
- `autowatch start <config_file>` - Start watcher with YAML config
- `autowatch stop` - Stop running watcher instances
- `autowatch status` - Show active watchers and queue stats
- `autowatch config validate <file>` - Validate config schema
- `autowatch config generate <template>` - Generate config skeleton
- `autowatch queue list` - List pending and failed tasks
- `autowatch queue retry <task_id>` - Retry failed task
- `autowatch metrics` - Show throughput and latency metrics

**Config file structure:**
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

## Work Process

1. **Initialization**
   ```bash
   autowatch config validate ./research-watcher.yaml
   autowatch start ./research-watcher.yaml
   ```

2. **Event Detection**
   - Uses `watchdog` to monitor filesystem events
   - Events are debounced per watcher config (`debounce_ms`)
   - File metadata collected: path, size, mtime, event type

3. **Filtering**
   - Apply `patterns` and `ignore_patterns` glob filters
   - Evaluate `condition` expressions (Python `eval` with safe namespace)
   - Files failing filters are logged and dropped

4. **Batching**
   - Accumulate events up to `batch_size` or `debounce_ms` timeout
   - Batch delivered to AI pipeline as single request

5. **AI Processing**
   - Load pipeline config (`ai_pipelines.<name>`)
   - Render prompt template with context (`file_path`, `file_content`, metadata)
   - Call OpenAI API with specified model and parameters
   - Parse `output_format` (json/text/markdown)

6. **Result Handling**
   - Store result in Redis with key: `autowatch:result:<hash(file_path)>`
   - If `archive_path` set, move processed file to archive
   - Publish event: `autowatch.completed` with result metadata

7. **Error Handling**
   - Retry up to `max_retries` on transient failures
   - After retries exhausted, mark task as `failed` and publish `autowatch.failed`
   - Failed tasks can be manually requeued with `autowatch queue retry`

8. **Metrics**
   - Prometheus metrics exposed on `:9090/metrics` if `metrics_enabled: true`
   - Tracks: `events_received`, `batches_processed`, `ai_calls_total`, `processing_duration_seconds`, `errors_total`

## Golden Rules

- Never run Auto Watcher as root; use dedicated `autowatch` user
- Always validate config before starting: `autowatch config validate`
- Set `AUTO_WATCHER_OPENAI_API_KEY` in `.env` file, never in config YAML
- Ensure `archive_path` is on same filesystem to avoid cross-device link errors
- For production, run with `systemd` unit and `Restart=on-failure`
- Do not point watchers at directories with >10k files; use specific subdirectories
- Always set `batch_size` appropriate to your AI model's rate limits
- Monitor `autowatch.failed` events; investigate >1% failure rate
- Use separate Redis DB (`db=1`) from other applications
- Keep AI pipeline prompts in version control, not embedded in config

## Examples

**Example 1: Watch papers directory**
```bash
# config: papers-watcher.yaml
autowatch config validate papers-watcher.yaml
autowatch start papers-watcher.yaml
```

Input file: `/data/papers/incoming/attention-is-all-you-need.pdf`
Event: Created
AI prompt rendered:
```
Summarize the following research paper, extracting: 1) Main contribution, 2) Methodology, 3) Results, 4) Limitations.

Paper: [Extracted text from PDF...]
```
Output (stored in Redis):
```json
{
  "file": "/data/papers/incoming/attention-is-all-you-need.pdf",
  "summary": "Introduces Transformer architecture...",
  "key_points": ["Self-attention", "No recurrence", "State-of-the-art on MT"],
  "timestamp": "2026-03-03T10:23:45Z"
}
```

**Example 2: Monitor experimental results**
```bash
autowatch queue list
# Output:
# ID    Status    Created              Pipeline
# a1b2c pending  2026-03-03T10:25:00Z experiment-analyzer
# d4e5f failed   2026-03-03T10:24:15Z experiment-analyzer

autowatch queue retry d4e5f
# Task requeued, new ID: x9y8z
```

**Example 3: Status check**
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

## Rollback Commands

**Stop watcher:**
```bash
autowatch stop
# Verifies: no watcher processes remain (`pgrep -f autowatch`)
```

**Restore archived files:**
```bash
# If archive_path is /data/processed/
find /data/processed/ -name "*.processed" -exec sh -c '
  mv "$1" "$(echo "$1" | sed "s|/data/processed/|/data/papers/incoming/|" | sed "s|\.processed$||")"
' _ {} \;
```

**Clear Redis queue (dangerous - use with caution):**
```bash
redis-cli -n 1 DEL autowatch:queue:*
redis-cli -n 1 DEL autowatch:results:*
```

**Revert config changes:**
```bash
# Assuming config in git
git checkout HEAD~1 -- ./research-watcher.yaml
# Validate and restart
autowatch config validate ./research-watcher.yaml && autowatch restart ./research-watcher.yaml
```

**Recover from corrupted Redis:**
```bash
# Flush specific DB and restart watcher
redis-cli -n 1 FLUSHDB
autowatch stop && autowatch start ./research-watcher.yaml
# Reprocess archived files manually if needed
```

**Disable specific watcher without stopping whole system:**
```bash
# Edit config, set enabled: false for that watcher
sed -i 's/enabled: true/enabled: false/' ./research-watcher.yaml
autowatch config validate ./research-watcher.yaml && autowatch reload ./research-watcher.yaml
```
```