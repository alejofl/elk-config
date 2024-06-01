# Monitoreo de servicios con Heartbeat

## Introducción

Heartbeat es una herramienta de monitoreo de infraestructura y servicios que permite a los equipos de operaciones monitorear la disponibilidad y el rendimiento de sus servicios. Heartbeat es parte del stack de Elastic.

El objetivo es monitorear cualquier servicio o servidor al que se le pueda enviar un request TCP o ICMP. Heartbeat puede ser configurado para monitorear servicios HTTP, HTTPS, TCP, ICMP, etc.

Tipicamente se instala Heartbeat como parte de un servicio de monitoreo que corre en un host separado y posiblemente incluso fuera de la red donde corren los servicios que se quieren monitorear.

## Instalación en Linux

Para instalar Heartbeat en Linux, primero necesitamos descargar el paquete de instalación. Para ello, podemos ir a la página de descargas de Elastic y copiar el link de descarga del paquete de Heartbeat.

```bash
curl -L -O https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-8.13.4-amd64.deb
```

Una vez descargado el paquete, lo instalamos con el siguiente comando:

```bash
sudo dpkg -i heartbeat-8.13.4-amd64.deb
```

## Configuración

La configuración de Heartbeat se realiza a través de un archivo de configuración llamado `heartbeat.yml`. Este archivo se encuentra en la carpeta `/etc/heartbeat`.

Lo importante de definir el output de Heartbeat. En principio, nosotros queremos que la información se envíe directamente a Elasticsearch. Para ello, debemos configurar el output de Heartbeat de la siguiente manera:

```yaml
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["10.1.102.178:9200", "10.1.103.227:9200"]
  # Performance preset - one of "balanced", "throughput", "scale", "latency", or "custom".
  preset: balanced
```

Se especifican los hosts de Elasticsearch a los que se enviarán los datos de monitoreo. En este caso, se están enviando los datos a dos nodos de Elasticsearch, de manera que si uno de ellos falla, el otro puede seguir recibiendo los datos.

## Monitores

Hay dos maneras para definir un monitor:

- Mediante la adición de un archivo `.yml` a la carpeta `/etc/heartbeat/monitors.d`.
- Mediante la adición de una sección en el archivo de configuración de Heartbeat. La sección a añadir es `heartbeat.monitors`. Luego, se despliega una lista de todos los monitores.

En ambos casos, el contenido es el mismo. A continuación, se muestra un ejemplo de un monitor HTTP:

```yaml
type: http
enabled: true
# ID used to uniquely identify this monitor in Elasticsearch even if the config changes
id: webserver-monitor
# Human readable display name for this service in Uptime UI and elsewhere
name: Webserver Monitor
# List of URLs to query
urls: ["http://10.1.1.254"]
# Configure task schedule
schedule: '@every 10s'
# Total test connection and data exchange timeout
timeout: 25s
```

En este caso, se está monitoreando un servidor web que corre en la dirección `http://10.1.1.254`. El monitor se ejecuta cada 10 segundos y el tiempo de timeout es de 25 segundos. Si el servidor web no responde en 25 segundos, el monitor lo considerará como caído.

## Iniciando el servicio

Para iniciar el servicio de Heartbeat, se utiliza el siguiente comando:

```bash
sudo systemctl start heartbeat
```

Para que el servicio se inicie automáticamente al encender la máquina, se utiliza el siguiente comando:

```bash
sudo systemctl enable heartbeat
```

## Configuración de Elasticsearch

Para que Heartbeat pueda enviar los datos a Elasticsearch, es necesario configurar los índices que se van a utilizar. Para ello, se puede utilizar el siguiente comando, ofrecido por Heartbeat:

```bash
heartbeat setup -e
```

## Visualización de los datos en Kibana

Heartbeat viene con dashboards y UIs preconstruidos para visualizar el estado de los servicios. Los dashboards están disponibles en el [repositorio de GitHub de uptime-contrib](https://github.com/elastic/uptime-contrib/blob/master/dashboards/7.x/http_dashboard.ndjson).

Para importar los dashboards a Kibana, se debe ir a la sección de Management y luego a la sección de Saved Objects.

> [!TIP]
> El archivo de configuración de Heartbeat utilizado en la demostración se encuentran en el repositorio, en `/files/linux/heartbeat.yml`.
