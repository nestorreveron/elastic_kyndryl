# **LABORATORIO DÍA 03 – TROUBLESHOOTING REAL EN CLUSTER DE 2 NODOS**

# ===========================================

## Objetivo del día 03

Aprender a diagnosticar y resolver errores comunes en pipelines de ingestión **usando un clúster distribuido**, 
donde los problemas pueden aparecer tanto en el nodo coordinador como en nodos data.

Los escenarios incluyen:

1. Permisos incorrectos en archivos de origen
2. Grok parsing errors
3. Mapeos conflictivos (conflicting field types)
4. Corte de flujo / Output error
5. Desfase de tiempo (timestamps incorrectos)

Este laboratorio se ejecuta **desde es-node-1**, pero los errores afectarán al clúster completo.

---

# =====================================================

# **0. Validación inicial: el clúster de 2 nodos está OK**

# =====================================================

```bash
curl -s http://localhost:9200/_cluster/health?pretty
curl -s http://localhost:9200/_cat/nodes?v
curl -s http://localhost:9200/_cat/shards?v
```

Debes ver:

* 2 nodos: es-node-1 y es-node-2
* 1 shard primario y 1 réplica para cada índice
* Status: green

---

# =============================================

# **1. ERROR 1 – Permisos en archivos de origen**

# =============================================

Simularemos un pipeline que intenta leer un archivo donde el usuario `elasticsearch` **NO tiene permisos**.

## Crear archivo con permisos incorrectos

```bash
sudo mkdir -p /var/log/demo
sudo touch /var/log/demo/app.log
sudo chmod 600 /var/log/demo/app.log
```

Como `elasticsearch` no puede leer el archivo, verás errores en logs del ingestion node.

## Crear pipeline de ingestión (simulación)

```bash
curl -X PUT "localhost:9200/_ingest/pipeline/perm-test" \
  -H 'Content-Type: application/json' -d '
{
  "description": "Pipeline que intenta leer archivo",
  "processors": [
    { "grok": { "field": "message", "patterns": ["%{WORD:level} %{GREEDYDATA:msg}"] } }
  ]
}'
```

Simular ingestión:

```bash
curl -X POST "localhost:9200/logs-perm/_doc?pipeline=perm-test" \
  -H 'Content-Type: application/json' -d '{"message": "INFO Demo"}'
```

No fallará aquí (porque no lee realmente el archivo), pero en un entorno real verías:

```
Permission denied /var/log/demo/app.log
```

### Solución (aplicable al clúster):

```bash
sudo chown elasticsearch:elasticsearch /var/log/demo/app.log
sudo chmod 644 /var/log/demo/app.log
```

---

# =====================================

# **2. ERROR 2 – Grok Parsing Problem**

# =====================================

Creamos un pipeline con una expresión grok **incorrecta**.

```bash
curl -X PUT "localhost:9200/_ingest/pipeline/grok-error" \
  -H 'Content-Type: application/json' -d '
{
  "processors": [
    { "grok": { "field": "message", "patterns": ["%{WORD:level} %{INVALID:msg}"] } }
  ]
}'
```

Probar ingestión:

```bash
curl -X POST "localhost:9200/logs-grok/_doc?pipeline=grok-error" \
  -H 'Content-Type: application/json' -d '{"message": "INFO Todo bien"}'
```

Respuesta esperada:

```json
{
  "error" : {
    "root_cause" : [
      { "type" : "grok_exception" }
    ]
  }
}
```

### Solución recomendada

Usar Kibana Grok Debugger (si tuvieras Kibana).
Desde CLI, corregimos:

```bash
curl -X PUT "localhost:9200/_ingest/pipeline/grok-error" \
  -H 'Content-Type: application/json' -d '
{
  "processors": [
    { "grok": { "field": "message", "patterns": ["%{WORD:level} %{GREEDYDATA:msg}"] } }
  ]
}'
```

---

# ========================================

# **3. ERROR 3 – Mapeos Conflictivos**

# ========================================

Este es un error clásico y afecta AL CLÚSTER COMPLETO, no solo a un nodo.

Caso típico:

* Index 1 envía `"status": "OK"` → tipo = text / keyword
* Index 2 envía `"status": 200"` → tipo = long

### Crear índice inicial “bueno”

```bash
curl -X PUT "localhost:9200/logs-mapping" \
  -H 'Content-Type: application/json' -d '
{
  "mappings": {
    "properties": {
      "status": { "type": "keyword" }
    }
  }
}'
```

Insertar documento:

```bash
curl -X POST "localhost:9200/logs-mapping/_doc" \
 -H 'Content-Type: application/json' -d '{"status": "OK"}'
```

### Insertar dato conflictivo

```bash
curl -X POST "localhost:9200/logs-mapping/_doc" \
 -H 'Content-Type: application/json' -d '{"status": 200}'
```

Respuesta esperada:

```json
"mapper_parsing_exception"
```

### Solución: definir un mapping claro

Cambiar `status` a:

```yaml
status.keyword -> estado textual
status_code -> número
```

---

# =============================================

# **4. ERROR 4 – Output Error / Connection Refused**

# =============================================

Simulamos que Fluent Bit / Filebeat no puede enviar datos al clúster.

Apagamos temporalmente Elasticsearch en es-node-2:

```bash
sudo systemctl stop elasticsearch
```

Cuando intentes enviar datos:

```bash
curl -X POST "localhost:9200/test-output/_doc" \
 -H 'Content-Type: application/json' -d '{"msg": "hola"}'
```

El coordinador (es-node-1) intentará escribir y puede recibir:

```
connection refused
failed shard
timeout
```

### Comprobar el estado distribuido:

```bash
curl localhost:9200/_cluster/health?pretty
curl localhost:9200/_cat/shards?v
```

Verás shards UNASSIGNED o clúster en YELLOW/RED.

### Solución:

```bash
sudo systemctl start elasticsearch
```

Cluster vuelve a GREEN automáticamente.

---

# ======================================

# **5. ERROR 5 – Desfase de Hora / Timestamps**

# ======================================

Simulamos logs con timestamps UTC incorrectos.

Enviar documento con hora equivocada:

```bash
curl -X POST "localhost:9200/logs-time/_doc" \
  -H 'Content-Type: application/json' -d '
{
  "timestamp": "2024-01-01T00:00:00Z",
  "msg": "evento"
}'
```

Luego simular evento local “ahora”:

```bash
curl -X POST "localhost:9200/logs-time/_doc" \
  -H 'Content-Type: application/json' -d '
{
  "timestamp": "2023-01-01T10:00:00-06:00",
  "msg": "evento local"
}'
```

En búsquedas por rango:

```bash
curl -X GET "localhost:9200/logs-time/_search" \
  -H "Content-Type: application/json" -d '
{
  "query": { "range": { "timestamp": { "gte": "now-1h" }}}
}'
```

Puede que no aparezca nada.

### Solución típica:

Agregar processor `date` para normalizar:

```bash
curl -X PUT "localhost:9200/_ingest/pipeline/normalize-date" \
  -H 'Content-Type: application/json' -d '
{
  "processors": [
    {
      "date": {
        "field": "timestamp",
        "formats": ["ISO8601"]
      }
    }
  ]
}'
```

---

# =====================================================

# **6. Validación final del clúster distribuido**

# =====================================================

```bash
curl localhost:9200/_cat/indices?v
curl localhost:9200/_cat/shards?v
curl localhost:9200/_cluster/health?pretty
```

Debes ver:

* cada índice con shard primario + réplica en nodos distintos
* cluster en verde
* todos los pipelines funcionando correctamente


