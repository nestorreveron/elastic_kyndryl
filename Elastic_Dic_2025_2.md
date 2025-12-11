La idea es que todo lo ejecutes desde **es-node-1** (o desde donde prefieras) usando `curl`. El segundo nodo se usará para **replicación y alta disponibilidad**, pero no necesitas lanzar comandos en ambos para este lab.

---

## 0. Prerrequisitos (muy rápido)

En **cada VM** ya deberías tener:

* Elasticsearch instalado y corriendo (`systemctl status elasticsearch`)
* `xpack.security.enabled: false` para simplificar
* Clúster de 2 nodos funcionando

Desde `es-node-1` valida:

```bash
curl http://localhost:9200/_cluster/health?pretty
curl http://localhost:9200/_cat/nodes?v
```

Debes ver 2 nodos y `status` idealmente `green`.

---

# LABORATORIO AVANZADO

## Creación de índice `products-advanced` y búsquedas complejas

### Vista general de lo que haremos

1. Crear índice `products-advanced` con:

   * **Analizador personalizado**
   * Campos **multi-field** (`text` + `keyword`)
   * Campos **nested** para arrays de objetos
2. Indexar documentos de ejemplo con arrays de objetos
3. Ejecutar consultas avanzadas:

   * `fuzzy` search
   * `match_phrase` con `slop`
   * búsqueda con `regexp`
   * `bool` query con filtros complejos
4. Agregaciones avanzadas:

   * `date_histogram`
   * `cardinality`

Todo con comandos listos para pegar.

---

## 1. Crear el índice `products-advanced` (settings + mappings)

En **es-node-1**:

```bash
curl -X PUT "http://localhost:9200/products-advanced" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1,
    "analysis": {
      "filter": {
        "spanish_stop": {
          "type":       "stop",
          "stopwords":  "_spanish_"
        },
        "spanish_stemmer": {
          "type": "stemmer",
          "language": "light_spanish"
        }
      },
      "analyzer": {
        "spanish_custom": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "asciifolding",
            "spanish_stop",
            "spanish_stemmer"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "spanish_custom",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "spanish_custom"
      },
      "category": {
        "type": "keyword"
      },
      "tags": {
        "type": "keyword"
      },
      "price": {
        "type": "double"
      },
      "available_from": {
        "type": "date"
      },
      "attributes": {
        "type": "nested",
        "properties": {
          "key":   { "type": "keyword" },
          "value": { "type": "keyword" }
        }
      },
      "variants": {
        "type": "nested",
        "properties": {
          "color": { "type": "keyword" },
          "size":  { "type": "keyword" },
          "stock": { "type": "integer" }
        }
      }
    }
  }
}
EOF
```

Comprueba el mapping:

```bash
curl "http://localhost:9200/products-advanced/_mapping?pretty"
```

---

## 2. Indexar documentos con arrays de objetos (nested)

Usaremos el **Bulk API** para meter varios productos de una vez.

```bash
curl -X POST "http://localhost:9200/products-advanced/_bulk" \
  -H "Content-Type: application/x-ndjson" \
  -d @- <<'EOF'
{ "index": { "_id": 1 } }
{ "name": "Portátil profesional 15 pulgadas", "description": "Portátil ligero para trabajo remoto y desarrollo", "category": "laptop", "tags": ["oficina","desarrollo","remoto"], "price": 1200.0, "available_from": "2024-01-10T10:00:00Z",
  "attributes": [
    { "key": "cpu", "value": "i7" },
    { "key": "ram", "value": "16GB" },
    { "key": "storage", "value": "512GB SSD" }
  ],
  "variants": [
    { "color": "gris", "size": "15", "stock": 5 },
    { "color": "negro", "size": "15", "stock": 2 }
  ]
}
{ "index": { "_id": 2 } }
{ "name": "Portátil gaming 17 pulgadas", "description": "Equipo para juegos intensivos y streaming", "category": "laptop", "tags": ["gaming","streaming"], "price": 2000.0, "available_from": "2024-02-05T12:00:00Z",
  "attributes": [
    { "key": "cpu", "value": "i9" },
    { "key": "ram", "value": "32GB" },
    { "key": "gpu", "value": "RTX 4080" }
  ],
  "variants": [
    { "color": "negro", "size": "17", "stock": 3 }
  ]
}
{ "index": { "_id": 3 } }
{ "name": "Monitor ultra ancho 34 pulgadas", "description": "Monitor curvo ideal para multitarea y observabilidad de dashboards", "category": "monitor", "tags": ["observabilidad","dashboard","oficina"], "price": 800.0, "available_from": "2023-11-20T09:00:00Z",
  "attributes": [
    { "key": "resolution", "value": "3440x1440" },
    { "key": "panel", "value": "IPS" }
  ],
  "variants": [
    { "color": "negro", "size": "34", "stock": 10 }
  ]
}
{ "index": { "_id": 4 } }
{ "name": "Teclado mecánico silencioso", "description": "Teclado mecánico silencioso para trabajo en oficina y turnos nocturnos", "category": "keyboard", "tags": ["oficina","silencioso"], "price": 150.0, "available_from": "2023-12-01T08:30:00Z",
  "attributes": [
    { "key": "switch", "value": "silent-red" },
    { "key": "layout", "value": "ES" }
  ],
  "variants": [
    { "color": "negro", "size": "full", "stock": 20 }
  ]
}
EOF
```

Variante curl si falla el anterior

```bash
curl --request POST \
  --url http://localhost:9200/products-advanced/_bulk \
  --header 'Content-Type: application/x-ndjson' \
  --data '{ "index": { "_id": 1 } }
{ "name": "Portátil profesional 15 pulgadas", "description": "Portátil ligero para trabajo remoto y desarrollo", "category": "laptop", "tags": ["oficina","desarrollo","remoto"], "price": 1200.0, "available_from": "2024-01-10T10:00:00Z", "attributes": [ { "key": "cpu", "value": "i7" }, { "key": "ram", "value": "16GB" }, { "key": "storage", "value": "512GB SSD" } ],  "variants": [ { "color": "gris", "size": "15", "stock": 5 }, { "color": "negro", "size": "15", "stock": 2 }  ]}
{ "index": { "_id": 2 } }
{ "name": "Portátil gaming 17 pulgadas", "description": "Equipo para juegos intensivos y streaming", "category": "laptop", "tags": ["gaming","streaming"], "price": 2000.0, "available_from": "2024-02-05T12:00:00Z",  "attributes": [    { "key": "cpu", "value": "i9" },    { "key": "ram", "value": "32GB" },    { "key": "gpu", "value": "RTX 4080" }  ],  "variants": [    { "color": "negro", "size": "17", "stock": 3 }  ]}
{ "index": { "_id": 3 } }
{ "name": "Monitor ultra ancho 34 pulgadas", "description": "Monitor curvo ideal para multitarea y observabilidad de dashboards", "category": "monitor", "tags": ["observabilidad","dashboard","oficina"], "price": 800.0, "available_from": "2023-11-20T09:00:00Z",  "attributes": [    { "key": "resolution", "value": "3440x1440" },    { "key": "panel", "value": "IPS" }  ],  "variants": [    { "color": "negro", "size": "34", "stock": 10 }  ]}
{ "index": { "_id": 4 } }
{ "name": "Teclado mecánico silencioso", "description": "Teclado mecánico silencioso para trabajo en oficina y turnos nocturnos", "category": "keyboard", "tags": ["oficina","silencioso"], "price": 150.0, "available_from": "2023-12-01T08:30:00Z",  "attributes": [    { "key": "switch", "value": "silent-red" },    { "key": "layout", "value": "ES" }  ],  "variants": [    { "color": "negro", "size": "full", "stock": 20 }  ]}
'
```

Verifica que hay documentos:

```bash
curl "http://localhost:9200/products-advanced/_count?pretty"
```

---

## 3. Consultas avanzadas

### 3.1. Fuzzy search (tolerancia a errores tipográficos)

Ejemplo: buscar “portatil profecional” con errores ortográficos.

```bash
curl -X GET "http://localhost:9200/products-advanced/_search" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "portatil profecional",
        "fuzziness": "AUTO"
      }
    }
  }
}
EOF
```

Deberías recuperar el documento con `Portátil profesional 15 pulgadas`.

---

### 3.2. Phrase query con `slop` (términos cercanos pero no exactos)

Buscar frases similares a “juegos intensivos streaming” en la descripción:

```bash
curl -X GET "http://localhost:9200/products-advanced/_search" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "query": {
    "match_phrase": {
      "description": {
        "query": "juegos intensivos streaming",
        "slop": 3
      }
    }
  }
}
EOF
```

`slop: 3` permite que las palabras estén separadas pero manteniendo proximidad.

---

### 3.3. Búsqueda con `regexp` (regex sobre campo keyword)

Ejemplo: buscar productos cuya categoría empiece por “lap” (como `laptop`):

```bash
curl -X GET "http://localhost:9200/products-advanced/_search" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "query": {
    "regexp": {
      "category": "lap.*"
    }
  }
}
EOF
```

Otro ejemplo: regex sobre `name.keyword` para nombres que contengan “Portátil”:

```bash
curl -X GET "http://localhost:9200/products-advanced/_search" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "query": {
    "regexp": {
      "name.keyword": ".*Portátil.*"
    }
  }
}
EOF
```

---

### 3.4. `bool` query con filtros complejos (incluyendo nested)

Caso típico:

* productos categoría `laptop`
* precio entre 1000 y 2500
* que tengan atributo `ram=32GB`
* y NO sean gaming

```bash
curl -X GET "http://localhost:9200/products-advanced/_search" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "query": {
    "bool": {
      "must": [
        { "term": { "category": "laptop" } }
      ],
      "filter": [
        {
          "range": {
            "price": {
              "gte": 1000,
              "lte": 2500
            }
          }
        },
        {
          "nested": {
            "path": "attributes",
            "query": {
              "bool": {
                "must": [
                  { "term": { "attributes.key": "ram" } },
                  { "term": { "attributes.value": "32GB" } }
                ]
              }
            }
          }
        }
      ],
      "must_not": [
        { "term": { "tags": "gaming" } }
      ]
    }
  }
}
EOF
```

Puedes cambiar las condiciones para jugar con performance y resultados.

---

## 4. Agregaciones avanzadas

### 4.1. `date_histogram` – ventas / productos por fecha de disponibilidad

Agrupemos los productos por mes de `available_from`:

```bash
curl -X GET "http://localhost:9200/products-advanced/_search" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "size": 0,
  "aggs": {
    "productos_por_mes": {
      "date_histogram": {
        "field": "available_from",
        "calendar_interval": "month",
        "format": "yyyy-MM"
      }
    }
  }
}
EOF
```

`size: 0` indica que no queremos documentos, solo resultados de agregación.

---

### 4.2. `cardinality` – cuántas categorías o tags únicas tenemos

```bash
curl -X GET "http://localhost:9200/products-advanced/_search" \
  -H "Content-Type: application/json" \
  -d @- <<'EOF'
{
  "size": 0,
  "aggs": {
    "categorias_unicas": {
      "cardinality": {
        "field": "category"
      }
    },
    "tags_unicos": {
      "cardinality": {
        "field": "tags"
      }
    }
  }
}
EOF
```

Esto devuelve una aproximación del número de categorías y tags distintos (HyperLogLog).

---

## 5. ¿Dónde entra el segundo nodo de Azure en este lab?

Aunque trabajas todo desde `es-node-1`, el segundo nodo:

* almacena réplicas de los shards
* participa en consultas (búsquedas distribuidas)
* asegura alta disponibilidad

Puedes verlo con:

```bash
curl "http://localhost:9200/_cat/shards/products-advanced?v"
```

Deberías ver:

* un shard primario (`p`) en un nodo
* una réplica (`r`) en el otro

Eso demuestra que el lab se ejecuta sobre un **clúster distribuido real en Azure**, no en un nodo aislado.
