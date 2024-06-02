# Using Logstash on Windows

To obtain Windows events, we will use a tool called `winlogbeat`. The configuration of the latter does not allow obtaining logs from a specific folder but has a set of predetermined `event logs`.

Given that there may be times when you want to obtain logs stored in a specific folder and `winlogbeat` does not allow it, we will also explain the `filebeat` tool, which offers this functionality.

We will start by explaining the configuration of `winlogbeat`, then `filebeat`, and finally `logstash`.

> [!NOTE]
> In all three cases, we will be working in the folder `C:\program-version-number`.

## Installing `winlogbeat`

1. Download the `.zip` file of `winlogbeat` from the website:
    
    [Winlogbeat quick start: installation and configuration | Winlogbeat Reference [8.13] | Elastic](https://www.elastic.co/guide/en/beats/winlogbeat/current/winlogbeat-installation-configuration.html)
    
2. Unzip the `.zip` file and place it in the corresponding folder so that it is located at `C:\winlogbeat-8.13.4`.
3. Open a terminal as an administrator, navigate to the `winlogbeat` folder `C:\winlogbeat-8.13.4`, and run `.\install-service-winlogbeat.ps1`. This installs the program as a service on the computer.
    
    ```text
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
    
    If it doesn't work, try this:
    
    ```text
    C:\winlogbeat> powershell -ExecutionPolicy Bypass -File .\install-service-winlogbeat.ps1
    ```
    
4. Configure `winlogbeat` so that it:
   * Can catch Windows events.
   * Sends everything it collects to `logstash`.
   * Performs logging.
    
    > [!NOTE]
    > In the following link, you can see more available configurations to obtain logs of other types.
    > [Configure Winlogbeat | Winlogbeat Reference [8.13] | Elastic](https://www.elastic.co/guide/en/beats/winlogbeat/current/configuration-winlogbeat-options.html)
    
    In the specific options section, we specify the `event_logs` we want to catch:
      * `Application`: Application events.
      * `System`: System events.
      * `Security`: Security events.
      * `Microsoft-Windows-Sysmon/Operational`: Sysmon events, an advanced event monitoring utility.
      * `Windows PowerShell`: Events related to Windows PowerShell with specific event IDs (400, 403, 600, 800).
      * `Microsoft-Windows-PowerShell/Operational`: PowerShell events with specific event IDs (4103, 4104, 4105, 4106).
      * `ForwardedEvents`: Forwarded events from other computers.
    
    ```yaml
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
    
    In the outputs section, we specify where to send the caught `event_logs`. In this case, we will send them to `logstasg` on port `localhost:5044`:
    
    ```yaml
    # ================================== Outputs ===================================

    # ------------------------------ Logstash Output -------------------------------
    output.logstash:
      # The Logstash hosts
      hosts: ["localhost:5044"]
    ```
    
    In the processors section, we configure two processors:

      * `add_host_metadata`: Adds information such as hostname, IP address, unique host ID, operating system name, and system architecture. However, in this case, the processor will only run if the event does not contain the forwarded tag.
      * `add_cloud_metadata`: Adds information about the cloud instance where the host is running, such as cloud provider, region, instance ID, among other details. The `~` indicates that this processor will run on all events without any additional conditions.

    ```yaml
    # ================================= Processors =================================
    processors:
    - add_host_metadata:
        when.not.contains.tags: forwarded
    - add_cloud_metadata: ~
    ```
    
    In the logging section, we specify the following to create log files in the corresponding folder:
    
    ```yaml
    # ================================== Logging ===================================

    logging.to_files: true
    logging.files:
      path: C:\winlogbeat-8.13.4\logs
    logging.level: info
    ```
    
5. In the terminal that we opened previously as administrators, run the following command to test if the configuration file is correctly formatted:
    
    ```text
    C:\winlogbeat> .\winlogbeat.exe test config -c .\winlogbeat.yml -e
    ```
    
    It should return:
    
    ```text
    C:\winlogbeat> .\winlogbeat.exe test config -c .\winlogbeat.yml -e
    ...
    Config OK
    ```
    
6. Before starting the service, we need to install `logstash` and `filebeat`.

## Installing `filebeat`

1. Download the `.zip` file of `filebeat` from the website:
    
    [Filebeat quick start: installation and configuration | Elastic](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation-configuration.html)
    
2. Unzip the `.zip` file and place it in the corresponding folder so that it is located at `C:\filebeat-8.13.4`.

3. Configure `filebeat` to obtain logs from a specific folder. In this case, we will obtain logs related to Windows Firewall:
   
   First, configure the inputs. We will obtain logs from the folder `C:\Windows\System32\LogFiles\Firewall\pfirewall.log`, and for that, we must configure the `type` as `filestream`, enable it with `enabled: true`, and set the `id` configuration.

   ```yaml
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

    In this section, we configure the `filebeat` modules. We specify the path so that it can find them. In our case, we won't have any extra modules, but we will keep the default configuration.

    ```yaml
    # ============================== Filebeat modules ==============================

    filebeat.config.modules:
    # Glob pattern for configuration loading
    path: ${path.config}/modules.d/*.yml

    # Set to true to enable config reloading
    reload.enabled: false

    # Period on which files under path should be checked for changes
    #reload.period: 10s
    ```

    In this section, we configure the output, that is, where to send all the collected logs. In this case, we will send them to `localhost:5045` so that `logstash` can catch them on that port.

    ```yaml
    # ================================== Outputs ===================================

    # Configure what output to use when sending the data collected by the beat.

    # ------------------------------ Logstash Output -------------------------------
    output.logstash:
    # The Logstash hosts
    hosts: ["localhost:5045"]
    ```

    In this section, we configure the processing of the information. This came by default in this way, and it will be kept like this.
    
    ```yaml
    # ================================= Processors =================================
    processors:
    - add_host_metadata:
        when.not.contains.tags: forwarded
    - add_cloud_metadata: ~
    - add_docker_metadata: ~
    - add_kubernetes_metadata: ~
   ```

4. Finally, with the configuration done, we proceed to run the `install-service-filebeat` program as administrators, which is located in the `C:\filebeat-8.13.4` folder. To do this, simply right-click and select `Run with PowerShell`, which will open a Windows PowerShell tab and start its execution. Alternatively, we can open a terminal as administrators in the corresponding folder (`C:\filebeat-8.13.4`) and run the following command:

    ```text
    C:\filebeat-8.13.4> .\install-service-filebeat.ps1
    ```

    > [!NOTE]
    > If script execution is disabled on your system, you'll need to configure the execution policy of the current session to allow the script to run. To do this, run the following command in the terminal:
    >    ```text
    >    C:\filebeat-8.13.4> PowerShell.exe -ExecutionPolicy UnRestricted -File .\install-service-filebeat.ps1`.
    >    ````

## Installing `Logstash`

1. Download `Logstash`, unzip the folder, and save it in `C:\`. 
2. Navigate to the `C:\logstash-8.13.4\config` folder and set the `logstash` configuration file, which is `logstash.yml`.
    ```yaml
    path.data: C:/logstash-8.13.4/data
    pipeline.workers: 2
    pipeline.batch.size: 125
    config.reload.automatic: true
    http.host: "127.0.0.1"
    http.port: 9600-9700
    ```
3. In the same folder `C:\logstash-8.13.4\config`, we'll create two pipeline configuration files. One for the `winlogbeat` event pipeline and the other for `filebeat` logs. These files are separated into three sections, `input`, `filter`, and `output` (their names are self-explanatory).

    > [!IMPORTANT]
    > In this case, we'll explain the configuration used for the EC2 Windows instance. If used locally, only the IPs in the `output` need to be modified. Below, you'll find an example for this configuration type.

    We'll start with the `winlogbeat` pipeline configuration file named `pipeline-events.conf`. First, we set the port through which the information enters, and we also assign a type to the logs, which in this case will be `windows`. For the output, we'll send the logs to the `elasticsearch` cluster with the corresponding IPs, and we also add an `index` that will characterize all Windows events sent from `winlogbeat`.

    ```text
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

    The next file is the `filebeat` pipeline configuration file named `pipeline-firewall.conf`. We'll see that for the input, we set the same port as the output port in `filebeat`, and we assign a `type` of log `firewall`. Then, we create a filter to only take those logs that match the presented regular expression. Finally, in the output, we configure it to send the logs to the same place as in the previous pipeline, but with the distinction that now the `index` depends on the `type` of log `firewall`.

    ```text
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

    If we want to test that we can obtain Windows events in a local environment, we need to create a `pipeline-events.conf` file as follows:

    ```text
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

    Note that for this case, a security section was added. We have the `user` and `password` configured in the [`installing-elasticsearch-and-kibana-in-windows.md`](./installing-elasticsearch-and-kibana-in-windows.md) file.

4. Now we proceed to configure the `pipelines.conf` file present in the `C:\logstash-8.13.4\config` folder. Here we'll specify the pipeline configuration files to be used by `logstash`.

    ```yaml
    - pipeline.id: pipeline-windows-events
      path.config: "/logstash-8.13.4/config/pipeline-events.conf"
    - pipeline.id: pipeline-windows-firewall
      path.config: "/logstash-8.13.4/config/pipeline-firewall.conf"
    ```
5. Download NSSM from the page:

    [NSSM - the Non-Sucking Service Manager](https://nssm.cc/download)

6. Extract the `nssm.exe` file from the `nssm-<version>\win64\nssm.exe` folder and place it in `C:\logstash-8.13.4\bin\`.

7. Open a terminal as administrator and validate the pipeline configuration files by running the following command:

    ```text
    C:\logstash-8.13.4> .\bin\logstash.bat -t -f C:\logstash-8.13.4\config\pipeline-events.conf
    C:\logstash-8.13.4> .\bin\logstash.bat -t -f C:\logstash-8.13.4\config\pipeline-firewall.conf
    ```

    It should return the following, specifically with `Configuration OK`:

    ```text
    "Using bundled JDK: C:\logstash-8.13.4\jdk\bin\java.exe"
    C:/logstash-8.13.4/vendor/bundle/jruby/3.1.0/gems/concurrent-ruby-1.1.9/lib/concurrent-ruby/concurrent/executor/java_thread_pool_executor.rb:13: warning: method redefined; discarding old to_int
    ...
    [2024-05-20T18:39:14,819][INFO ][logstash.runner          ] Using config.test_and_exit mode. Config Validation Result: OK. Exiting Logstash
    ```

8. Open a terminal as administrators and run the following command:

    ```bash
    C:\logstash-8.13.4> .\bin\nssm.exe install logstash
    ```

9. Running the above command will bring up a `NSSM service installer` window. In it, we need to specify the following parameters:
    - Path (path to `logstash.bat`): `C:\logstash-8.13.4\bin\logstash.bat`.
    - Startup Directory (path to the `bin` directory): `C:\logstash-8.13.4\bin`.
    - Arguments: leave it empty.

        ![instaling-elk-on-windows-start-service](../img/instaling-elk-on-windows-nssm.png)
        
    Click **Install service** to create the service.

10. Open a terminal as administrators and run the `logstash` and `winlogbeat` services (the `filebeat` service is already started):

    ```text
    C:> Start-Service logstash
    C:> Start-Service winlogbeat
    ```
