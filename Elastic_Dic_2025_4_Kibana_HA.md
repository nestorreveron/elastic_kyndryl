# 🧪 LAB – Alta Disponibilidad de Kibana (HA) con 2 nodos + Azure Load Balancer

## Objetivo

Desplegar **dos instancias de Kibana** (una en `es-node-1` y otra en `es-node-2`) detrás 
de un **Azure Load Balancer (LB)** en el puerto 5601, de forma que:

* Los usuarios acceden siempre vía **IP pública del LB**
* Si una instancia de Kibana cae, la otra sigue atendiendo
* Los dashboards, visualizaciones y data views son compartidos (mismo `.kibana` index)
* Tu lab de Día 4 funciona igual, pero en **modo HA real**

---

## A. Variables iniciales en CloudShell

Ajusta solo el Resource Group y, si hace falta, los nombres de las VMs:

```bash
RG="kyndrylXX"        # Resource group del lab
LOC="eastus2"
VM1="es-node-1"
VM2="es-node-2"

KIB_LB="kibana-lb"
KIB_BE_POOL="kibana-bepool"
KIB_PROBE="kibana-probe"
KIB_RULE="kibana-rule"
KIB_PORT=5601

# (Opcional) Si recuerdas el NSG:
NSG="nsg-elastic-lab"
```

---

## B. Crear IP pública y Load Balancer para Kibana (HA)

### 1. IP pública del LB

```bash
az network public-ip create \
  -g $RG \
  -n ${KIB_LB}-pip \
  --sku Standard \
  --allocation-method static
```

### 2. Crear el Load Balancer

```bash
az network lb create \
  -g $RG \
  -n $KIB_LB \
  --sku Standard \
  --frontend-ip-name kib-frontend \
  --backend-pool-name $KIB_BE_POOL \
  --public-ip-address ${KIB_LB}-pip
```

### 3. Health Probe para Kibana

Kibana escucha en 5601, así que usamos un probe TCP a ese puerto:

```bash
az network lb probe create \
  -g $RG \
  --lb-name $KIB_LB \
  -n $KIB_PROBE \
  --protocol tcp \
  --port $KIB_PORT
```

### 4. Regla de balanceo de carga (frontend 5601 → backend 5601)

Con persistencia de sesión por IP de cliente (básica para UX más estable):

```bash
az network lb rule create \
  -g $RG \
  --lb-name $KIB_LB \
  -n $KIB_RULE \
  --frontend-ip-name kib-frontend \
  --backend-pool-name $KIB_BE_POOL \
  --protocol Tcp \
  --frontend-port $KIB_PORT \
  --backend-port $KIB_PORT \
  --probe-name $KIB_PROBE \
  --idle-timeout 30 \
  --enable-tcp-reset true \
  --load-distribution SourceIP
```

(`SourceIP` = afinidad simple de sesión, suficiente para el lab.)

---

## C. Asociar las NIC de es-node-1 y es-node-2 al Backend Pool

### 1. Obtener NICs de las VMs

```bash
NIC1_ID=$(az vm show -g $RG -n $VM1 --query "networkProfile.networkInterfaces[0].id" -o tsv)
NIC2_ID=$(az vm show -g $RG -n $VM2 --query "networkProfile.networkInterfaces[0].id" -o tsv)

NIC1=$(basename "$NIC1_ID")
NIC2=$(basename "$NIC2_ID")
```

### 2. Agregar NICs al Backend Pool

```bash
az network nic ip-config address-pool add \
  -g $RG \
  --address-pool $KIB_BE_POOL \
  --ip-config-name ipconfig1 \
  --nic-name $NIC1 \
  --lb-name $KIB_LB

az network nic ip-config address-pool add \
  -g $RG \
  --address-pool $KIB_BE_POOL \
  --ip-config-name ipconfig1 \
  --nic-name $NIC2 \
  --lb-name $KIB_LB
```

---

## D. NSG – Permitir acceso al puerto 5601 (Kibana)

Si tu NSG es `nsg-elastic-lab` y ya lo usas en ambas NIC/Subnet:

```bash
az network nsg rule create \
  -g $RG \
  --nsg-name $NSG \
  -n Allow-Kibana-5601 \
  --priority 1030 \
  --access Allow \
  --protocol Tcp \
  --direction Inbound \
  --source-address-prefixes Internet \
  --destination-port-ranges $KIB_PORT
```

Con esto, el tráfico desde Internet al puerto 5601 (sobre la IP del LB) será aceptado hacia las VMs.

---

## E. Instalar Kibana en ambos nodos (es-node-1 y es-node-2)

Primero obtén las IP públicas para conectarte:

```bash
az vm list-ip-addresses -g $RG -o table
```

Conéctate a `es-node-1`:

```bash
ssh elasticadmin@IP_PUBLICA_ES_NODE_1
```

En cada VM (repetir en `es-node-1` y `es-node-2`):

```bash
sudo apt update
sudo apt install -y kibana
```

---

## F. Configuración HA de Kibana en ambos nodos

La clave aquí es que **ambos kibana.yml sean prácticamente idénticos**, apuntando al mismo cluster y con las mismas encryption keys.

En *cada nodo*:

```bash
sudo nano /etc/kibana/kibana.yml
```

Configura (ajusta IP privadas de ES a las tuyas):

```yaml
server.host: "0.0.0.0"
server.port: 5601

# URL pública que verá el usuario (IP del Load Balancer)
server.publicBaseUrl: "http://<IP_PUBLICA_KIBANA_LB>:5601"

elasticsearch.hosts: ["http://10.10.1.4:9200","http://10.10.1.5:9200"]

# Claves de encripción – DEBEN SER IGUALES EN TODAS LAS INSTANCIAS KIBANA
xpack.security.enabled: false

xpack.encryptedSavedObjects.encryptionKey: "MiClaveSuperLargaDe32CaracteresMinimo_1"
xpack.reporting.encryptionKey: "MiClaveSuperLargaDe32CaracteresMinimo_2"
xpack.security.encryptionKey: "MiClaveSuperLargaDe32CaracteresMinimo_3"
```

Puntos importantes:

* Cambia `<IP_PUBLICA_KIBANA_LB>` por la IP del LB (la sacamos en el siguiente paso).
* Usa strings largos (32+ chars) para las keys, pero **que sean exactamente las mismas en ambos nodos**.
* `elasticsearch.hosts` debe apuntar al mismo cluster que ya tienes configurado para los labs 1–3.

Guarda el archivo y repite exactamente la misma configuración en `es-node-2`.

---

## G. Habilitar y arrancar Kibana en ambos nodos

En cada nodo:

```bash
sudo systemctl enable kibana
sudo systemctl restart kibana
sudo systemctl status kibana
```

Ver que el servicio queda en `active (running)`.

---

## H. Obtener IP pública del Load Balancer de Kibana

Volvemos a CloudShell:

```bash
az network public-ip show \
  -g $RG \
  -n ${KIB_LB}-pip \
  --query "ipAddress" -o tsv
```

Supongamos que devuelve:

```text
52.X.Y.Z
```

Esa será la URL de acceso para TODOS tus alumnos / usuarios:

```text
http://52.X.Y.Z:5601
```

---

## I. Repetir el LAB DEL DÍA 4, pero ahora en HA

Ahora, todo lo que hacías en tu **lab original del Día 4**, lo harás **entrando por el Load Balancer**, no por las IP individuales de las VMs.

### 1. Acceder a Kibana (HA)

En el navegador:

```text
http://52.X.Y.Z:5601
```

### 2. Crear el Data View (antes Index Pattern)

Menú: **Stack Management → Data Views → Create data view**

* Data view:
  `products-advanced*`
  o `logs-demo*`
* Campo de tiempo:
  `@timestamp` o `available_from`

### 3. Discover

Menú: **Discover**

* Seleccionas tu data view
* Ajustas el rango de tiempo
* Verificas documentos, campos, etc.

### 4. Visualizaciones clave

Creas exactamente las mismas:

* Pie: Errores HTTP
* Métrica: Latencia promedio / price
* Bar: HTTP codes vs category
* Lens: Top productos

### 5. Dashboard

Armas el dashboard:

* Añades las visualizaciones
* Ajustas layout
* Guardas como:
  `Dashboard Día 4 – Observabilidad Elasticsearch (HA)`

### 6. KQL y filtros

Pruebas:

```kql
category : "laptop"
price > 1500
status >= 400
attributes.value : "32GB"
price < 20 or price > 3000
name : portátil
```

Verificas que:

* Las visualizaciones se actualizan dinámicamente
* El panel responde a rangos de tiempo
* Los filtros afectan a todo el dashboard

### 7. Exportar a PDF

En el dashboard:

* **Share → Generate PDF → Optimize for printing**
* Descargas el PDF y verificas contenido correcto.

---

## J. Prueba de FAILOVER de Kibana (HA real)

Esta parte es la que les encanta a los alumnos “enterprise”.

1. Entra a Kibana vía LB:
   `http://52.X.Y.Z:5601`

2. Navega al dashboard del Día 4.

3. Ahora, en **es-node-1**, detén Kibana:

   ```bash
   sudo systemctl stop kibana
   ```

4. Refresca el navegador:

   * Sigues entrando al mismo dashboard
   * El tráfico ahora lo maneja **Kibana en es-node-2**
   * El usuario no necesita cambiar de URL

5. Vuelve a arrancar Kibana en es-node-1:

   ```bash
   sudo systemctl start kibana
   ```

Puedes hacer la misma prueba a la inversa (parar Kibana en es-node-2).

---

## K. Checklist final del LAB Kibana HA

Se considera que el laboratorio está **completado** cuando:

* [ ] Existe un Load Balancer Standard en Azure para Kibana (puerto 5601).
* [ ] Las NIC de `es-node-1` y `es-node-2` están en el backend pool.
* [ ] Kibana está instalado y corriendo en ambos nodos.
* [ ] `kibana.yml` es prácticamente idéntico en ambos nodos (mismas keys y misma URL de ES).
* [ ] Puedes acceder a Kibana solo por la IP del LB.
* [ ] El lab del Día 4 (data view, Discover, visualizaciones, dashboard, KQL, PDF) funciona entrado por el LB.
* [ ] Si paras Kibana en un nodo, el dashboard sigue disponible sin cambiar la URL.


