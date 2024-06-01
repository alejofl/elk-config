# Utilizar Logstash en Ubuntu

Para poder obtener los eventos de Windows se utilizará una herramienta denominada `winlogbeat`. La configuración de esta última no permite obtener los logs presentes en una carpeta determinada, sino que tiene una serie de `event logs` determinadas.

Dado que es posible que en algún momento se quiera obtener los logs que se almacenan en una carpeta determinada y `winlogbeat` no lo permite, también se explicará la herramienta `filebeat` que ofrece esta funcionalidad.

Iniciaremos explicando la configuración de `winlogbeat`, luego la de `filebeat` y finalmente la de `logstash`. 

> [!IMPORTANT]
> En la presente configuración dispondremos a todos los programas en la misma carpeta que es la siguiente: `C:\program-version`.

## Instalando `winlogbeat`

1. Descargamos de la página el `.zip` de `winlogbeat`
    
    [Winlogbeat quick start: installation and configuration | Winlogbeat Reference [8.13] | Elastic](https://www.elastic.co/guide/en/beats/winlogbeat/current/winlogbeat-installation-configuration.html)
    
2. Descomprimimos el archivo `.zip` y lo ubicamos en la carpeta correspondiente para que quede en `C:\winlogbeat-8.13.4`.
3. Abrimos una terminal como administrador, nos ubicamos en la carpeta de `winlogbeat` `C:\winlogbeat-8.13.4` y corremos `.\install-service-winlogbeat.ps1`. Con esto logramos que se instale el programa como servicio en la computadora.
    
    ```bash
    C:\winlogbeat> .\install-service-winlogbeat.ps1
    Security warning
    Run only scripts that you trust. While scripts from the internet can be useful,
    this script can potentially harm your computer. If you trust this script, use
    the Unblock-File cmdlet to allow the script to run without this warning message.
    Do you want to run C:\Program Files\Winlogbeat\install-service-winlogbeat.ps1?
    [D] Do not run  [R] Run once  [S] Suspend  [?] Help (default is "D"): R
    
    Status   Name               DisplayName
    ------   ----               -----------
    Stopped  winlogbeat         winlogbeat
    ```
    
    Si no funciona intenta esto:
    
    ```bash
    C:\winlogbeat> powershell -ExecutionPolicy Bypass -File .\install-service-winlogbeat.ps1
    ```
    
4. Configuramos a `winlogbeat` para que:
   * Pueda catchear los eventos de Windows.
   * Para que todo lo que recopile lo envíe a `logstash`. 
   * Para que realice logging.
    
    <aside>
    
    **Leer en el siguiente link las configuraciones para poder obtener logs de diferentes tipos. Con la configuración actual solamente obtendremos los logs de aplicaciones y los de seguridad.**
    
    [Configure Winlogbeat | Winlogbeat Reference [8.13] | Elastic](https://www.elastic.co/guide/en/beats/winlogbeat/current/configuration-winlogbeat-options.html)
    
    </aside>
    
    En el apartado de specific options especificamos los `event_logs` que queremos catchear:
      * `Application`: registros de eventos de la aplicación.
      * `System`: registros de eventos del sistema.
      * `Security`: registros de eventos de seguridad.
      * `Microsoft-Windows-Sysmon/Operational`: registros de eventos de Sysmon, una utilidad de monitorización de eventos avanzada.
      * `Windows PowerShell`: registros de eventos relacionados con Windows PowerShell con ciertos IDs de eventos especificados (400, 403, 600, 800)
          * 400: Este evento se genera cuando un script de PowerShell se inicia.
          * 403: Se registra cuando un script de PowerShell se detiene.
          * 600: Este evento se produce cuando se inicia un nuevo proceso de PowerShell.
          * 800: Se registra cuando se ejecuta un comando en PowerShell (puede incluir tanto comandos de script como comandos interactivos).
      * `Microsoft-Windows-PowerShell/Operational`: registros de eventos de PowerShell con IDs de eventos específicos (4103, 4104, 4105, 4106).
          * 4103: Este evento se genera cuando un módulo de PowerShell se carga.
          * 4104: Se registra cuando se descarga un módulo de PowerShell.
          * 4105: Este evento se produce cuando se comienza a ejecutar un script de PowerShell.
          * 4106: Se registra cuando se finaliza la ejecución de un script de PowerShell.
      * `ForwardedEvents`: registros de eventos reenviados de otros equipos.
    
    ```bash
      # ======================== Winlogbeat specific options =========================

      winlogbeat.event_logs:
        - name: Application
          ignore_older: 72h

        - name: System

        - name: Security

        - name: Microsoft-Windows-Sysmon/Operational

        - name: Windows PowerShell
          event_id: 400, 403, 600, 800

        - name: Microsoft-Windows-PowerShell/Operational
          event_id: 4103, 4104, 4105, 4106

        - name: ForwardedEvents
          tags: [forwarded]
    ```
    
    En el apartado de outputs especificamos a dónde es que enviaremos los `event_logs` catcheados. En este caso los enviaremos a `logstasg` en el puerto `localhost:5044`:
    
    ```bash
      # ================================== Outputs ===================================
    
      # ------------------------------ Logstash Output -------------------------------
      output.logstash:
        # The Logstash hosts
        hosts: ["localhost:5044"]
    ```
    En el apartado de processors configuraremos dos procesadores:

      * `add_host_metadata`: agrega información como el nombre del host, la dirección IP, el ID único del host, el nombre del sistema operativo y la arquitectura del sistema. Sin embargo, en este caso, el procesador solo se ejecutará si el evento no contiene la etiqueta forwarded.
      * `add_cloud_metadata`: agrega información sobre la instancia de la nube en la que se está ejecutando el host, como el proveedor de la nube, la región, el ID de instancia, entre otros detalles. El ~ indica que este procesador se ejecutará en todos los eventos sin ninguna condición adicional.

    ```bash
    # ================================= Processors =================================
    processors:
    - add_host_metadata:
        when.not.contains.tags: forwarded
    - add_cloud_metadata: ~
    ```
    
    En el apartado de Logging especificamos lo siguiente que sirve para poder crear archivos de logs en la carpeta correspondiente:
    
    ```bash
    # ================================== Logging ===================================
    
    logging.to_files: true
    logging.files:
      path: C:\winlogbeat-8.13.4\logs
    logging.level: info
    ```
    
5. En la terminal que habíamos abierto previamente como administradores, corremos el siguiente comando para testear que el archivo de configuración esté correctamente formado:
    
    ```bash
    C:\winlogbeat> .\winlogbeat.exe test config -c .\winlogbeat.yml -e
    ```
    
    Nos tiene que devolver:
    
    ```bash
    C:\winlogbeat> .\winlogbeat.exe test config -c .\winlogbeat.yml -e
    ...
    Config OK
    ```
    
6. Antes de poder iniciar el servicio debemos instalar `logstash` y `filebeat`.

## Instalando `filebeat`

1. Descargamos de la página el `.zip` de `filebeat`
    
    [Filebeat quick start: installation and configuration | Elastic](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation-configuration.html)
    
2. Descomprimimos el archivo `.zip` y lo ubicamos en la carpeta correspondiente para que quede en `C:\filebeat-8.13.4`.

3. Configuramos a `filebeat` para que obtenga los logs de una carpeta en específico. En este caso obtendremos los logs relacionados con Windows Firewall:
   
   Primero configuramos los inputs. Veamos que obtendremos los logs presentes en la carpeta `C:\Windows\System32\LogFiles\Firewall\pfirewall.log` y para ello debemos configurar al `type` en `filestream`, habilitarla con `enabled: true` y setearle la configuración de `id`.
   ```bash
    # ============================== Filebeat inputs ===============================

    filebeat.inputs:
    # filestream is an input for collecting log messages from files.
    - type: filestream

    # Unique ID among all inputs, an ID is required.
    id: my-filestream-id

    # Change to true to enable this input configuration.
    enabled: true

    # Paths that should be crawled and fetched. Glob based paths.
    paths:
        # - /var/log/*.log
        - C:\Windows\System32\LogFiles\Firewall\pfirewall.log
    ```

    En esta sección se configuran los `filebeat` modules. Le especificamos el path para que los pueda encontrar. En nuestro caso no tendremos ningún módulo extra, pero mantendremos la configuración default.

    ```bash
    # ============================== Filebeat modules ==============================

    filebeat.config.modules:
    # Glob pattern for configuration loading
    path: ${path.config}/modules.d/*.yml

    # Set to true to enable config reloading
    reload.enabled: false

    # Period on which files under path should be checked for changes
    #reload.period: 10s
    ```
    En este apartado configuramos el output, es decir, a dónde enviará todos los logs recolectados. En este caso lo enviaremos a `localhost:5045` para que los pueda catchear `logstash` en ese puerto.

    ```bash
    # ================================== Outputs ===================================

    # Configure what output to use when sending the data collected by the beat.

    # ------------------------------ Logstash Output -------------------------------
    output.logstash:
    # The Logstash hosts
    hosts: ["localhost:5045"]
    ```
    En esta sección configuramos al igual que con `winlogbeat` la forma en la que se va a procesar la información. Esto vino por default de esta manera y se lo mantendrá así.
    ```bash
    # ================================= Processors =================================
    processors:
    - add_host_metadata:
        when.not.contains.tags: forwarded
    - add_cloud_metadata: ~
    - add_docker_metadata: ~
    - add_kubernetes_metadata: ~
   ```
4. Finalmente, con la configuración realizada procedemos a correr el programa `install-service-filebeat` como administradores que se encuentra en la carpeta `C:\filebeat-8.13.4`. Para ello solamente debemos hacer click derecho y clickear en `Run with powershell`, esto abrirá una pestaña de Windows Powershell e iniciará su ejecución. También podemos abrir una terminal como administradores en la carpeta correspondiente (`C:\filebeat-8.13.4`) y corrmos el siguiente comando:

    ```bash
    C:\filebeat-8.13.4> .\install-service-filebeat.ps1
    ```
> [!NOTE]
> Si la ejecución de scripts está desactivada en su sistema, deberá configurar la política de ejecución de la sesión actual para permitir que se ejecute la secuencia de comandos. Para ello debe correr el siguiente comando en la terminal:
>    ```bash
>    C:\filebeat-8.13.4> PowerShell.exe -ExecutionPolicy UnRestricted -File .\install-service-filebeat.ps1`.
>    ````

## Instalando `logstash`

1. Descargamos `logstash`, descomprimimos la carpeta y la guardamos en `C:\`. 
2. Ingresamos a la carpeta `C:\logstash-8.13.4\config` y seteamos el archivo de configuración de `logstash` que es `logstash.yml`. Aquí configuraremos la carpeta dónde se guardará la información, la cantidad de workers para los pipelines, el tamaño del batch, la IP del host y los puertos por dónde ingresa la información de `winlogbeat` y `filebeat`.
    
    ```bash
    path.data: C:/logstash-8.13.4/data
    pipeline.workers: 2
    pipeline.batch.size: 125
    config.reload.automatic: true
    http.host: "127.0.0.1"
    http.port: 9600-9700
    ```
3. En la misma carpeta `C:\logstash-8.13.4\config`, crearemos dos archivos de configuración de pipelines. Uno para el pipeline de eventos de `winlogbeat` y el otro para los logs de `filebeat`. Veremos que estos archivos se encuentran separados en tres secciones, `input`, `filter` y `output` (sus nombres son autoexplicativos). En este caso explicaremos la configuración utilizada para la instancia EC2 de Windows. Si se llega a utilizar de manera local entonces solamente se debe modificar las IPs presentes en el `output`.
  
    Iniciaremos por el archivo de configuración de pipeline de `winlogbeat` que se denomina `pipeline-events.conf`. En primer lugar, seteamos el puerto por el que ingresa la información y también le asignamos un type a los logs que en este caso será `windows`. Para lo que es el output enviaremos los logs al cluster de `elasticsearch` que tienen las IPs correspondientes y además le agregamos un `index` que es lo que caracterizará a todos los eventos de Windows que se enviarán desde `winlogbeat`.
        
    ```bash
    input {
        beats {
            port => 5044
            type => "windows"
        }
    }

    output {
        elasticsearch {
            hosts => ["http://10.1.102.178:9200", "http://10.1.103.227:9200"]
            index => "%{[type]}-%{+YYYY.MM.dd}"
        }
    }
    ```
    El siguiente archivo es el archivo de configuración de pipeline de `filebeat` que se denomina `pipeline-firewall.conf`. Veamos que para el input pusimos el mismo puerto que el puerto de salida en `filebeat` y le asignamos un `type` de log `firewall`. Luego creamos un filter para solamente tomar aquellos logs que coincidan con la expresión regular presentada. Por último, en el output configuramos para que envíe los logs al mismo lugar que en el pipeline anterior, pero con la distinción de que ahora el `index` depende del `type` de log `firewall`.

    ```bash
    input {
        beats {
            port => 5045
            type => "firewall"
        }
    }

    filter {
        grok {
            match => { "message" => "%{DATESTAMP:timestamp} %{WORD:action} %{WORD:protocol} %{IPORHOST:src_ip} %{IPORHOST:dst_ip} %{NUMBER:src_port} %{NUMBER:dst_port} %{NUMBER:size} %{GREEDYDATA:flags}" }
        }
    }

    output {
    elasticsearch {
        hosts => ["http://10.1.102.178:9200", "http://10.1.103.227:9200"]
        index => "%{[type]}-%{+YYYY.MM.dd}"
    }
    }
    ```

    Si queremos probar solamente la parte de eventos de windows en ambiente local debemos configurar el archivo `pipeline-events.conf` de la siguiente manera:

    ```bash  
    input {
        beats {
        port => 5044
        }
    }

    filter {
        if [event_data] {
        mutate {
            rename => { "[event_data][param1]" => "message" }
        }
        }
        date {
        match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
        }
    }

    output {
        if [@metadata][pipeline] {
            elasticsearch {
            hosts => ["http://localhost:9200"]
            manage_template => false
            index => "%{[@metadata][beat]}-%{[@metadata][version]}" 
            action => "create" 
            pipeline => "%{[@metadata][pipeline]}" 
            user => "johndoe"
            password => "123456"   
            }
        } else {
            elasticsearch {
            hosts => ["http://localhost:9200"]
            manage_template => false
            index => "%{[@metadata][beat]}-%{[@metadata][version]}" 
            action => "create"
            user => "johndoe"
            password => "123456" 
            }
        }
    }
    ```
    Veamos que para este caso se agregó un apartado de seguridad. Tenemos el `user` y `password` que configuramos en el archivo `installing-elasticsearch-and-kibana-in-windows.md`.

4. Ahora procedemos a configurar el archivo de `pipelines.conf` presente en la carpeta `C:\logstash-8.13.4\config`. En él especificaremos los archivos de configuración de pipeline a utilizar por `logstash`.

    ```bash
    - pipeline.id: pipeline-windows-events
    path.config: "/logstash-8.13.4/config/pipeline-events.conf"
    - pipeline.id: pipeline-windows-firewall
    path.config: "/logstash-8.13.4/config/pipeline-firewall.conf"
    ```
5. Descargamos NSSM de la página:
    
    [NSSM - the Non-Sucking Service Manager](https://nssm.cc/download)
    
6. Extraemos el archivo `nssm.exe` de la carpeta  `nssm-<version>\win64\nssm.exe` y lo ubicamos en `C:\logstash-8.13.4\bin\`.
   
7. Abrimos una terminal como administrador y balidamos el archivo de configuración del pipeline corriendo el siguiente comando:
    
    ```bash
    C:\logstash-8.13.4> .\bin\logstash.bat -t -f C:\logstash-8.13.4\config\logstash-sample.conf
    ```
    
    Nos tiene que devolver lo siguiente, específicamente con el `Configuration OK`:
    
    ```bash
    "Using bundled JDK: C:\logstash-8.13.4\jdk\bin\java.exe"
    C:/logstash-8.13.4/vendor/bundle/jruby/3.1.0/gems/concurrent-ruby-1.1.9/lib/concurrent-ruby/concurrent/executor/java_thread_pool_executor.rb:13: warning: method redefined; discarding old to_int
    ...
    [2024-05-20T18:39:14,819][INFO ][logstash.runner          ] Using config.test_and_exit mode. Config Validation Result: OK. Exiting Logstash
    ```
    
8. Abrimos una terminal como administradores y corremos el siguiente comando:
    
    ```bash
    C:\logstash-8.13.4 > .\bin\nssm.exe install logstash
    ```
    
9.  Corriendo el comando anterior aparece una ventana de `NSSM service installer`. En ella debemos especificar los siguientes parámetros:
    - Path (path a `logstash.bat`): `C:\logstash-8.13.4\bin\logstash.bat`.
    - Startup Directory (path al directorio `bin`): `C:\logstash-8.13.4\bin`.
    - Arguments: lo dejamos vacío.
        
        ![instaling-elk-on-windows-start-service](../img/instaling-elk-on-windows-nssm.png)
        
    
    Le damos a **Install service** para que se cree el servicio.
    
    
10. Abrimos una terminal como administradores y corremos los servicios de `logstash` y `winlogbeat` (el servicio de `filebeat` ya lo iniciamos):
    
    ```bash
    C:> Start-Service logstash
    C:> Start-Service winlogbeat
    ```