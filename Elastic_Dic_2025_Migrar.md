# Migración de 7.17 a 8.15

### 🟦 Cluster Elasticsearch

| Nodo      | IP privada | Versión | Roles              | Kibana |
| --------- | ---------- | ------- | ------------------ | ------ |
| es-node-1 | 10.10.1.4  | 7.17.29 | master/data/ingest | **Sí** |
| es-node-2 | 10.10.1.5  | 7.17.29 | master/data/ingest | No     |
| es-node-3 | 10.10.1.6  | 7.17.29 | master/data/ingest | No     |

### 🟦 Kibana HA (parcial)

Actualmente **solo hay Kibana en es-node-1**, por lo tanto:

* El Load Balancer **no aplica aún**
* Debemos instalar Kibana en **es-node-2** luego del upgrade
* Luego reconstruimos correctamente el HA con las claves de encripción

### 🟦 Índices

Todos tus índices tienen:

```txt
pri: 1
rep: 1
```

Excelente: **esto permite rolling upgrade real sin pérdida de datos.**

### 🟦 Versión destino

**Elasticsearch / Kibana 8.15.0 (estable y recomendada)**

---

# 🟥 **PARTE 1 – Preparación y snapshot (obligatorio)**

### 1.1. Verificar estado real del cluster

En es-node-1:

```bash
curl -s http://localhost:9200/_cluster/health?pretty
curl -s http://localhost:9200/_cat/nodes?v
curl -s http://localhost:9200/_cat/indices?v
```

Debes ver:

```txt
status: "green"
number_of_nodes: 3
```

---

### 1.2. Crear un repositorio de snapshots en los tres nodos

### 1) Crear carpeta física

Ejecuta esto en **es-node-1, es-node-2 y es-node-3**:

```bash
sudo mkdir -p /var/lib/elasticsearch/backups
sudo chown -R elasticsearch:elasticsearch /var/lib/elasticsearch/backups
```

### 2) Configurar repositorio

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

```yml
# Path a repositorio de backups
path.repo: [/var/lib/elasticsearch/backups]
```

### 3) Registrar repositorio

```bash
curl -X PUT "http://10.10.1.4:9200/_snapshot/backup_repo" -H "Content-Type: application/json" -d '
{
  "type": "fs",
  "settings": {
    "location": "/var/lib/elasticsearch/backups"
  }
}'
```

### 4) Crear snapshot completo

```bash
curl -X PUT "http://10.10.1.4:9200/_snapshot/backup_repo/preupgrade?wait_for_completion=true"
```

Debe devolver:

```txt
"state": "SUCCESS"
```

### 1.3. (Opcional) Crear snapshot de la máquina virtual en Azure

Acceder a Inicio > Resource Manager > Grupo de recursos > `kyndryl0X` > `es-node-X_OsDisk_X_xxxxxxxxx` > Crear instantánea

> - Nombre = nodoX_snap_previoupgrade
> - Tipo de instantánea = completa

Repetir en los tres nodos

### 🟦 ¿Qué logramos hasta aquí?

✔ Tienes respaldo completo

✔ Puedes restaurar si falla algo

✔ Índices con réplica → seguro hacer rolling upgrade

✔ Listo para migración

---

# 🟥 **PARTE 2 – Rolling Upgrade de Elasticsearch 7.17 → 8.15 (3 nodos)**

Regla de oro:

**Nunca parar más de un nodo a la vez.**

El orden recomendado según Elastic es:

1. Nodos data/ingest
2. Nodos master elegidos
3. Supervisar cluster
4. Repetir

Como tus nodos cumplen todos los roles, lo hacemos así:

➡ Primero **es-node-1**
➡ Luego **es-node-2**
➡ Último **es-node-3**

---

# 🟩 **NODO 1 — es-node-1 (10.10.1.4)**

## 2.1. Parar servicio

```bash
sudo systemctl stop elasticsearch
```

---

## 2.2. Instalar Elasticsearch 8.15.0

### 1) Importar nuevamente la clave GPG (válida para 8.x)

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic-8.gpg
```

### 2) Crear el repositorio 8.x

```bash
echo "deb [signed-by=/usr/share/keyrings/elastic-8.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### 3) Instalación nueva versión

```bash
sudo apt update
sudo apt install elasticsearch=8.15.0
```

Si el repo no tiene pinning, instalar última 8.x:

```bash
sudo apt install elasticsearch
```

---

## 2.3. Ajustar elasticsearch.yml

Abre el archivo:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Reemplaza/asegura:

```yaml
cluster.name: kyndryl-elk-lab

node.name: es-node-1
node.roles: [ master, data, ingest ] # En 8 cambia la forma de especificar los roles

network.host: 0.0.0.0
http.port: 9200

discovery.seed_hosts: ["10.10.1.4","10.10.1.5","10.10.1.6"]

cluster.initial_master_nodes: ["es-node-1","es-node-2","es-node-3"]

xpack.security.enabled: false
xpack.security.transport.ssl.enabled: false #Esta propiedad se añade respecto de lo configurado en v7
```

---

## 2.4. Iniciar servicio

```bash
sudo systemctl start elasticsearch
```

---

## 2.5. Verifica nodo actualizado

```bash
curl -s http://10.10.1.4:9200/
```

Debe mostrar:

```
"number" : "8.15.0"
```

---

## 2.6. Verifica cluster green

```bash
curl -s http://10.10.1.4:9200/_cluster/health?pretty
```

Debe quedar:

```
status: "green"
number_of_nodes: 3
```

**No avances a nodo 2 hasta que el cluster esté green.**

> **Nota:** Podrían observarse cluster en yellow por la diferencia de versión entre nodos
---

# 🟩 **NODO 2 — es-node-2 (10.10.1.5)**

**Mismos pasos exactos:**

1. Stop servicio
2. Instalar ES 8.15.0
3. Editar elasticsearch.yml
4. Start
5. Verificar health green

---

# 🟩 **NODO 3 — es-node-3 (10.10.1.6)**

Mismos pasos.

---

# 🟩 **Resultado esperado del cluster**

Ejecutar:

```bash
curl -s http://localhost:9200/_cat/nodes?v
```

Debe mostrar algo como:

```bash
ip        heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.10.1.5           18          73   0    0.01    0.02     0.00 dim       *      es-node-2
10.10.1.6           35          69   0    0.00    0.00     0.00 dim       -      es-node-3
10.10.1.4           11          65   0    0.00    0.02     0.00 dim       -      es-node-1
```

Y health:

```bash
curl -s http://localhost:9200/_cluster/health?pretty
```

```bash
"status" : "green"
```

---

# 🟥 PARTE 3 – Kibana HA + Upgrade 7.17 → 8.15

Actualmente:

Si ya agregaste el repo 8.x al instalar Elasticsearch 8.x:

deb https://artifacts.elastic.co/packages/8.x/apt stable main

Entonces no necesitas hacer nada extra, porque Elasticsearch y Kibana usan el mismo repositorio.
* es-node-1 → Kibana 7.17
* es-node-2 → **No tiene Kibana, salvo que hayas realizado el laboratorio Elastic_Dic_2025_4_Kibana_HA**
* es-node-3 → **No tiene Kibana**

Queremos completar un HA real: dos/tres instancias idénticas detrás del LB.

---

# 🟩 3.1 — Actualizar Kibana en es-node-1

```bash
sudo systemctl stop kibana
sudo apt update
sudo apt install kibana=8.15.0
```

---

# 🟩 3.2 — Instalar Kibana desde cero en es-node-2 y es-node-3

```bash
sudo apt update
sudo apt install kibana=8.15.0
```

---

# 🟩 3.3 — Configurar HA (con claves idénticas)

En **cada nodo**:

```bash
sudo nano /etc/kibana/kibana.yml
```

Configura:

```yaml
server.port: 5601
server.host: "0.0.0.0"
server.publicBaseUrl: "http://<IP-publica-del-LB>:5601"

elasticsearch.hosts: ["http://10.10.1.4:9200","http://10.10.1.5:9200","http://10.10.1.6:9200"]

# Enables you to specify a file where Kibana stores log output.
logging:
  appenders:
    file:
      type: file
      fileName: /var/log/kibana/kibana.log
      layout:
        type: json
  root:
    appenders:
      - default
      - file

# Specifies the path where Kibana creates the process ID file.
pid.file: /run/kibana/kibana.pid

#xpack.security.enabled: false
#xpack.encryptedSavedObjects.encryptionKey: "SuperClaveLargaDe32CaracteresMinimo123"
#xpack.reporting.encryptionKey: "OtraClaveLargaDe32CaracteresMinimo456"
#xpack.security.encryptionKey: "ClaveDiferenteLargaDe32CaracteresMinimo789"
```

**IMPORTANTE:** Las **3 claves deben ser EXACTAMENTE iguales** en todos los Kibana.
> **Nota:** Al actualizar kibana a versión 8 no arranca al configurar las propiedades de seguridad indicadas --> Las comentamos si las habiamos configurado en el lab de Kibana HA

---

# 🟩 Paso 3.4 — Iniciar Kibana en los nodos

```bash
sudo systemctl enable kibana
sudo systemctl restart kibana
sudo systemctl status kibana
```

Debe quedar ambos:

```
active (running)
```

> **Nota:** Es posible que kibana no arranque en los nodos en los que ya estaba instalada la versión 7 antes de instalar la versión 8 debido a cambios en el servicio relacionados con los parámetros de inicio del proceso. En este caso realizar la siguiente modificación.

```bash
sudo nano /etc/systemd/system/kibana.service
```

```bash
#ExecStart=/usr/share/kibana/bin/kibana --logging.dest="/var/log/kibana/kibana.log" --pid.file="/run/kibana/kibana.pid" --deprecation.skip_depr>
ExecStart=/usr/share/kibana/bin/kibana --pid.file="/run/kibana/kibana.pid"
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana
sudo systemctl start kibana
sudo systemctl status kibana
```

---

# 🟩 Paso 3.5 — Crear Load Balancer (Añadir nodos al LB)

La ejecución de este paso es obligatoria si deseas HA real en kibana, pero su ejecución depende de si previamente ya habias creado el Load Balancer para kibana o no.

- Si no habias generado el LB puedes crearlo siguiendo las instrucciones del **"Elastic_Dic_2025_4_Kibana_HA"**
- Si ya habias generado el LB sólo necesitas añadir al balanceo los nuevos nodos que hayas instalado de cero

Para añadir el nodo 3 al balanceo:
```bash
RG="kyndrylXX"        # Resource group del lab
VM3="es-node-3"
KIB_LB="kibana-lb"
KIB_BE_POOL="kibana-bepool"

NIC3_ID=$(az vm show -g $RG -n $VM3 --query "networkProfile.networkInterfaces[0].id" -o tsv)
NIC3=$(basename "$NIC3_ID")

az network nic ip-config address-pool add \
  -g $RG \
  --address-pool $KIB_BE_POOL \
  --ip-config-name ipconfig1 \
  --nic-name $NIC3 \
  --lb-name $KIB_LB
```

# 🟩 Paso 3.6 — Validar HA con Load Balancer

En tu navegador:

```
http://<IP-publica-del-LB>:5601
```

Deberás ver:

* Kibana 8.x
* Sin errores
* Todos los dashboards siguen funcionando

Luego prueba failover:

### Apaga Kibana del nodo 1:

```bash
sudo systemctl stop kibana
```

Refresca Kibana vía LB → debe seguir funcionando (nodos 2 y 3 responden).

---

# 🟥 PARTE 4 — Validaciones finales

Ejecuta:

```bash
curl -s http://localhost:9200/_cluster/health?pretty
curl -s http://localhost:9200/_cat/indices?v
curl -s http://localhost:9200/_cat/nodes?v
```

En Kibana:

* Discover
* Visualizations
* Dashboard
* Lens
* TSVB
* Reporting PDF

Todo debe funcionar mejor que antes.

---

# 🟥 PARTE 5 — Rollback si algo fallara

## Restaurar snapshot:

```bash
curl -X POST "http://localhost:9200/_snapshot/backup_repo/preupgrade/_restore"
```

## Downgrade:

```bash
sudo apt install elasticsearch=7.17.29
sudo apt install kibana=7.17.29
```

---
---

# 🟦 LAB COMPLETO — KIBANA 8.x PARA OPERACIONES & SEGURIDAD

---

Incluye:

✔ Crear índices desde cero

✔ Ingestar datos (logs, métricas y JSON personalizados)

✔ Crear Data Views

✔ Discover avanzado

✔ Visualizaciones (Lens, TSVB, Metric, Pie, Bar, Heatmap)

✔ Dashboards completos

✔ Creación de usuarios + roles (seguridad nativa de ES 8)

✔ Control de acceso por índice

✔ Alertas nativas en Kibana

✔ Exportación de dashboards

✔ Bonus: laboratorio de KQL y filtros globales

Todo **totalmente compatible con Kibana 8.15.x**.




# 🟩 **1. Crear índice desde cero (via API)**

Conéctate a cualquier nodo:

```bash
curl -X PUT "http://localhost:9200/app-logs" \
  -H 'Content-Type: application/json' \
  -d '{
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service":    { "type": "keyword" },
        "status":     { "type": "integer" },
        "message":    { "type": "text" }
      }
    }
  }'
```

Verificar:

```bash
curl -X GET http://localhost:9200/_cat/indices?v
```

---

# 🟩 **2. Insertar datos (logs simulados)**

```bash
curl -X POST "http://localhost:9200/app-logs/_bulk" \
  -H 'Content-Type: application/json' \
  -d '
{"index":{}}
{"@timestamp":"2025-12-12T12:00:00Z","service":"auth","status":200,"message":"Login success"}
{"index":{}}
{"@timestamp":"2025-12-12T12:01:00Z","service":"auth","status":401,"message":"Invalid password"}
{"index":{}}
{"@timestamp":"2025-12-12T12:02:00Z","service":"payments","status":500,"message":"DB timeout"}
{"index":{}}
{"@timestamp":"2025-12-12T12:03:00Z","service":"orders","status":200,"message":"Order created"}
{"index":{}}
{"@timestamp":"2025-12-12T12:04:00Z","service":"auth","status":403,"message":"Unauthorized"}
{"index":{}}
{"@timestamp":"2025-12-12T12:05:00Z","service":"payments","status":500,"message":"DB timeout"}
{"index":{}}
{"@timestamp":"2025-12-12T12:05:10Z","service":"orders","status":200,"message":"Order created"}
{"index":{}}
{"@timestamp":"2025-12-12T12:05:15","service":"payments","status":200,"message":"Payment processed"}
'
```

---

# 🟩 **3. Crear Data View en Kibana**

1. Entrar a kibana vía LB:

```
http://<LB-IP>:5601
```

2. Ir a:

```txt
Stack Management → Data Views → Create Data View
```

3. Crear:

* Nombre: **app-logs**
* Pattern: **app-logs***
* Campo de tiempo: **@timestamp**

Guardar.

---

# 🟩 **4. DISCOVER — análisis inicial**

En Discover:

1. Seleccionar el Data View `app-logs`
2. Filtrar:

```kql
status >= 400
```

3. Campos útiles:

* service
* status
* message
* @timestamp

4. Ordenar por timestamp
5. Añadir columnas personalizadas
6. Exportar resultados (CSV)

---

# 🟩 **5. VISUALIZACIONES — LENS**

## 5.1. **Errores por minuto (Time Series)**

Lens:

* Arrastrar @timestamp → Horizontal axis
* Arrastrar status → Vertical axis
* Filtrar:

```kql
status >= 400
```

Cambiar tipo: **Area / Line**

Guardar como:

```txt
Errores por minuto
```

---

## 5.2. **Top servicios por volumen (Bar chart)**

Lens:

* Arrastrar `service` → Horizontal axis
* Arrastrar `Records` → Vertical axis
* Ordenar: Descending

Guardar:

```txt
Top servicios por volumen
```

---

## 5.3. **Distribución de códigos HTTP (Pie)**

Lens:

* Arrastrar `status` → Slice
* Arrastrar `Records` → Size

Guardar:

```
Distribución HTTP status
```

---

## 5.4. **Latencia promedio (Metric visualization) — índice nuevo**

Creamos índice para métricas:

```bash
curl -X PUT "http://localhost:9200/metrics-demo" \
  -H 'Content-Type: application/json' \
  -d '{
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "service":    { "type": "keyword" },
        "latency_ms": { "type": "float" }
      }
    }
  }'
```

Insertar datos:

```bash
curl -X POST "http://localhost:9200/metrics-demo/_bulk" \
  -H 'Content-Type: application/json' \
  -d '
{"index":{}}
{"@timestamp":"2025-12-12T12:00:00Z","service":"auth","latency_ms":120}
{"index":{}}
{"@timestamp":"2025-12-12T12:00:00Z","service":"payments","latency_ms":450}
{"index":{}}
{"@timestamp":"2025-12-12T12:00:00Z","service":"orders","latency_ms":1200}
{"index":{}}
{"@timestamp":"2025-12-12T12:10:00Z","service":"auth","latency_ms":100}
{"index":{}}
{"@timestamp":"2025-12-12T12:10:00Z","service":"payments","latency_ms":400}
{"index":{}}
{"@timestamp":"2025-12-12T12:10:00Z","service":"orders","latency_ms":1100}
{"index":{}}
{"@timestamp":"2025-12-12T12:20:00Z","service":"auth","latency_ms":90}
{"index":{}}
{"@timestamp":"2025-12-12T12:20:00Z","service":"payments","latency_ms":500}
{"index":{}}
{"@timestamp":"2025-12-12T12:20:00Z","service":"orders","latency_ms":2000}
{"index":{}}
{"@timestamp":"2025-12-12T12:30:00Z","service":"auth","latency_ms":150}
{"index":{}}
{"@timestamp":"2025-12-12T12:30:00Z","service":"payments","latency_ms":520}
{"index":{}}
{"@timestamp":"2025-12-12T12:30:00Z","service":"orders","latency_ms":1900}
{"index":{}}
{"@timestamp":"2025-12-12T12:40:00Z","service":"auth","latency_ms":170}
{"index":{}}
{"@timestamp":"2025-12-12T12:40:00Z","service":"payments","latency_ms":490}
{"index":{}}
{"@timestamp":"2025-12-12T12:40:00Z","service":"orders","latency_ms":600}
'
```

Crear DataView: `metrics-demo*`

Crear visualización tipo **metric**:

* Metric: **Average latency_ms**
* Breakdown by: service

Guardar:

```
Latencia promedio por servicio
```

---

# 🟩 **6. Dashboard completo**

Crear dashboard:

Nombre:

```
Observabilidad – Día 4 Kibana 8.x
```

Agregar:

* Errores por minuto
* Top servicios por volumen
* Distribución de códigos HTTP
* Latencia promedio por servicio

Luego:

✔ Ajustar layout

✔ Filtros globales

✔ Range time picker

✔ Guardar dashboard

---

# 🟩 **7. Laboratorio de KQL para alumnos**

En Discover o Dashboard → Add filter.

1. Logs de un servicio:

```
service : "auth"
```

2. Errores:

```
status >= 400
```

3. Texto:

```
message : "timeout"
```

4. Expresión OR:

```
service : ("auth" or "payments")
```

5. Combinación:

```
service : "payments" and status >= 500
```

---

# 🟩 **8. ACTIVAR SEGURIDAD EN ELASTICSEARCH 8 (opcional, recomendado para lab)**

En ES 8.x la seguridad está activada por defecto cuando no la desactivas.

Si tienes actualmente:

```yaml
xpack.security.enabled: false
```

Puedes activar roles y usuarios:

---

## 8.1. Generar la Autoridad de Certificación para el cluster

Desde uno de los nodos

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil ca
```

1. Cuando pregunte, acepta el nombre por defecto. Este fichero contendrá el certificado público para tu CA y la llave privada usada para firmar los certificados de cada nodo.

2. Cuando solicite una contraseña puedes dejarla en blanco al no tratarse de un despliegue de producción.

Se generará `/usr/share/elasticsearch/elastic-stack-ca.p12`

## 8.2 Generar certificado para los nodos

Genera un certificado y llave privada para los nodos del cluster usando la CA generada en el paso previo. Se usará para la comunicación entre nodos del cluster.

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
```

Deja las opciones por defecto, simplemente pulsando enter en las preguntas. Se generará `/usr/share/elasticsearch/elastic-certificates.p12`

Copia el `elastic-certificates.p12` en cada uno de los nodos en la ruta `/etc/elasticsearch/elastic-certificates.p12` y dale permisos para que el usuario con el que se ejecuta el proceso de elastic pueda leerlo.

```bash
sudo cp -p /usr/share/elasticsearch/elastic-certificates.p12 /etc/elasticsearch/elastic-certificates.p12

sudo chown root:elasticsearch /etc/elasticsearch/elastic-certificates.p12
sudo chmod 660 /etc/elasticsearch/elastic-certificates.p12
```

## 8.3. Editar elasticsearch.yml en cada nodo

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

Quitar/descomentar y añadir:

```yaml
xpack.security.enabled: true

#Seguridad en comunicacion entre nodos. Requerido al habilitar la seguridad
xpack.security.transport.ssl.enabled: true

xpack.security.transport.ssl.verification_mode: certificate 
xpack.security.transport.ssl.client_authentication: required
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12

#La seguridad en http la mantenemos desactivada (habilitarla requiere pasos adicionales). Esta propiedad es necesaria para que funcione elasticsearch-reset-password
xpack.security.http.ssl.enabled: false
```
> **Nota:** Transport SSL must be enabled if security is enabled. Please set [xpack.security.transport.ssl.enabled] to [true] or disable security by setting [xpack.security.enabled] to [false]; for more information see [https://www.elastic.co/guide/en/elasticsearch/reference/8.15/bootstrap-checks-xpack.html#bootstrap-checks-tls]
>
>Server ssl configuration requires a key and certificate, but these have not been configured; you must set either [xpack.security.transport.ssl.keystore.path], or both [xpack.security.transport.ssl.key] and [xpack.security.transport.ssl.certificate]
>
> **Ref:** https://www.elastic.co/guide/en/elasticsearch/reference/8.15/security-basic-setup.html#encrypt-internode-communication
>
> **Ref:** https://www.elastic.co/guide/en/elasticsearch/reference/8.15/security-basic-setup-https.html

## 8.4. (Opcional) Configura la contraseña de los certificados en cada nodo

En el caso de que hayas puesto una password al generar el certificado de los nodos, ejecuta lo siguiente para añadirla a la configuración:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password

sudo /usr/share/elasticsearch/bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

Reiniciar:

```bash
sudo systemctl restart elasticsearch
```

Verifica que elastic haya levantado

```bash
curl -s http://localhost:9200
```

Esperado:

```bash
"status":401
```

---

## 8.5. Inicializar passwords de usuarios nativos

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

Esperado:

```bash
Password for the [elastic] user successfully reset.
New value: kFpvEyXiMK=hHA+IMaVN
```

También puedes hacerlo para:

```txt
- kibana_system
- logstash_system
- beats_system
```

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system
```

Esperado

```bash
Password for the [kibana_system] user successfully reset.
New value: xxxxPASSWORDxxxx
```


Ahora podrás verificar el estado del cluster de este modo

```bash
curl -s http://elastic:PASSWORD@localhost:9200/_cluster/health?pretty
```

---

## 8.6. Configurar Kibana para autenticación

En `/etc/kibana/kibana.yml`:

```
elasticsearch.username: "kibana_system"
elasticsearch.password: "<PASSWORD>"
```

Reiniciar Kibana:

```bash
sudo systemctl restart kibana
```

Ahora kibana es capaz de comunicar con elastic después de haber habilitado la autenticación básica, y adicionalmente nos solicitará credenciales para poder acceder. En este momento para acceder a kibana podemos usar el usuario `elastic` al que le hemos establecido la contraseña previamente.

---

# 🟩 **9. Crear usuarios y roles personalizados**

En Kibana:

```
Stack Management → Security → Users
```

Ejemplo: usuario `analista1`

Rol `viewer`:

* Permiso `read` sobre índices: `app-logs*`, `metrics-demo*`
* Permiso `read` en Dashboard, Visualize, Discover
* Sin permisos de gestión

Crear usuario:

```
analista1 / Pass123!!
```

Probar login.

---

# 🟩 **10. Crear alertas nativas**

En Kibana:

```
Observability → Alerts
```

Ejemplo: alerta cuando errores > 5 por minuto.

Criterios:

* Rule type: **Elasticsearch query**
* Índice: app-logs
* Query:

```
status >= 500
```

* Threshold: Document count > 3
* Notify: email o log (simulado)

---

# 🟩 **11. Exportar dashboard (PDF o PNG)**

En el dashboard:

```
Share → Generate PDF
```

Validar que exporta correctamente.

> **Nota:** Esta posibilidad de exportar en PDF no parece estar disponible en 8.15.0

---

# 🟩 BONUS — LAB DE ANOMALÍAS (TSVB)

Crea visualización TSVB:

* Panel type: Time Series
* Metric: Count()
* Group by: service
* Enable: Annotations (detect peaks)

Guardar como:

```
Análisis de anomalías por servicio
```
