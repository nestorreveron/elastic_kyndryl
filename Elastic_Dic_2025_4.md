# ELASTICSEARCH – DÍA 4

# **Laboratorio Completo: Kibana, Visualizaciones, KQL y Exportación a PDF**

**Objetivo del día 4:**
Llevar los datos previamente indexados en Elasticsearch hacia *Kibana*, explorarlos, crear visualizaciones clave, construir un dashboard profesional, aplicar filtros con KQL y finalmente exportarlo como PDF.

---

# 0. **Prerrequisito – Kibana instalado**

En tus 2 nodos Elasticsearch **solo instalaste ES**. Así que necesitas Kibana en **uno de los nodos** (generalmente `es-node-1`).

Si aún no lo has instalado, aquí está el comando:

```bash
sudo apt install kibana -y
```

Editar kibana.yml:

```bash
sudo nano /etc/kibana/kibana.yml
```

Asegura:

```
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
```

Levantar Kibana:

```bash
sudo systemctl enable kibana
sudo systemctl start kibana
```

Acceso:

```
http://<IP_PUBLICA_ES-NODE-1>:5601
```

---

# **LABORATORIO DÍA 4 – Paso a Paso**

---

# ⭐ 1. Crear un Index Pattern (Index Pattern → Data Views)

1. Entra a Kibana.
2. Menú lateral:
   **Stack Management → Data Views → Create data view**
3. Nombre del data view:

```
products-advanced*
```

O si usas los logs del Día 1:

```
logs-demo*
```

4. Selecciona el campo de fecha:

```
@timestamp   (si existe)  
o  
available_from (para products-advanced)
```

5. Crear.

**Resultado esperado:** Kibana reconoce tu índice y permite explorarlo.

---

# ⭐ 2. Discover Logs – Exploración de Datos

1. Menú lateral → **Discover**
2. Selecciona el data view creado.
3. Ajusta el rango de tiempo (arriba a la derecha):

```
Last 7 days → Last 30 days → Today → etc.
```

4. Verifica que los documentos están llegando desde ambos nodos Elasticsearch:

En la tabla de Discover:

* Verás documentos indexados
* Puedes expandir uno y ver los campos analizados

---

# ⭐ 3. Crear Visualizaciones Clave

Crearemos **4 visualizaciones esenciales** para observabilidad:

1. **Errores HTTP**
2. **Latencia promedio**
3. **Códigos HTTP por categoría**
4. **Top aplicaciones / productos**

---

## ✔ Visualización 1: Errores HTTP (Pie Chart)

1. Menú → **Analytics → Visualize Library**
2. Crear nueva → **Pie**
3. Data view:
   `products-advanced*` o `logs-demo*`
4. Configurar:

* **Slice by** → `status`
* **Size** → Count

5. Guardar como:

```
Errores HTTP - Day 4
```

---

## ✔ Visualización 2: Latencia Promedio (Metric)

Si tienes logs con `latency` o similar, o si usas un campo numérico como `price` para simular latencia:

1. Crear nueva → **Metric**
2. Metric → Aggregation:

```
Average of latency
```

o:

```
Average of price
```

3. Guardar como:

```
Latencia Promedio
```

---

## ✔ Visualización 3: Códigos HTTP por Categoría (Bar Chart)

1. Crear → **Bar vertical**
2. X-axis:

```
category.keyword
```

3. Y-axis:

```
Count
```

4. Color rules opcionales.

Guardar como:

```
HTTP Codes vs Category
```

---

## ✔ Visualización 4: Top Aplicaciones / Productos

Usando `name.keyword`:

1. Crear → **Lens**
2. Drop `name.keyword` en horizontal
3. Drop `price` o `count` en vertical

Guardar como:

```
Top Productos
```

---

# ⭐ 4. Construcción del Dashboard

1. Menú → **Analytics → Dashboard**
2. Crear nuevo Dashboard
3. Añadir las visualizaciones creadas:

* Errores HTTP
* Latencia Promedio
* HTTP Codes vs Category
* Top Productos

4. Organizar el layout:

* Métrica arriba
* Gráficas al lado
* Tabla opcional abajo

5. Guardar como:

```
Dashboard Día 4 – Observabilidad Elasticsearch
```

---

# ⭐ 5. KQL y detección de anomalías

Kibana Query Language permite:

* Filtrar por campos
* Crear búsquedas complejas
* Identificar anomalías en datos

---

## ✔ Ejemplos prácticos de KQL

### 1. Filtrar laptops:

```
category : "laptop"
```

### 2. Filtrar precios mayores a 1500:

```
price > 1500
```

### 3. Filtrar errores HTTP:

```
status >= 400
```

### 4. Buscar productos con RAM de 32GB (nested):

```
attributes.value : "32GB"
```

### 5. Detección básica de anomalías (rangos inesperados):

```
price < 20 or price > 3000
```

### 6. Términos similares (con analyzer):

```
name : portátil
```

**Resultado esperado:** las visualizaciones del dashboard se actualizan dinámicamente según el filtro.

---

# ⭐ 6. Exportar Dashboard a PDF

1. Abre el Dashboard.
2. Arriba a la derecha:
   **Share → Generate PDF**
3. Selecciona:

```
Optimize for printing
```

4. Descargar PDF.

**Resultado esperado:**
Dashboard limpio, sin campos vacíos ni errores.

---

# ⭐ **Pruebas del laboratorio (Checklist oficial)**

Debe cumplirse lo siguiente:

### ✔ Los datos se ven en Discover

Index pattern funcionando.

### ✔ Las visualizaciones se actualizan con filtros

KQL + Dashboard integrado.

### ✔ El dashboard responde a rangos de tiempo

Kibana refresca datos según fecha.

### ✔ KQL filtra correctamente

Debe reducir resultados con precisión.

### ✔ El PDF exportado no tiene inconsistencias

Visualizaciones completas y alineadas.


