專案說明
=======

這個專案提供了一個使用 Docker 部署 OpenTelemetry 的範例，旨在幫助開發者參考如何建置 OpenTelemetry 的 Docker 監控方案。範例特別是針對 Nginx 伺服器的監控和追蹤。

專案結構
-------

以下是專案中的主要文件，並對每個文件的內容進行詳細說明：

### 1. `Dockerfile.nginx`

這個 Dockerfile 用於建立一個包含 OpenTelemetry 模組的 Nginx 映像。

**內容說明：**

- **基礎映像：** 使用 `nginx:1.23.1` 作為基礎映像。
- **安裝必要工具：** 更新套件清單並安裝 `unzip`。
- **下載 OpenTelemetry WebServer SDK：** 從 GitHub 下載對應版本的 SDK 到 `/opt` 目錄。
- **解壓並安裝 SDK：** 在 `/opt` 目錄下解壓縮並安裝 SDK，執行 `install.sh` 腳本。
- **設置環境變數：** 將 SDK 的庫路徑添加到 `LD_LIBRARY_PATH`。
- **更新 Nginx 配置：** 在 `/etc/nginx/nginx.conf` 中載入 OpenTelemetry 模組。

```nginx
# https://opentelemetry.io/blog/2022/instrument-nginx/
FROM nginx:1.23.1
RUN apt-get update ; apt-get install unzip
ADD https://github.com/open-telemetry/opentelemetry-cpp-contrib/releases/download/webserver%2Fv1.0.3/opentelemetry-webserver-sdk-x64-linux.tgz /opt
RUN cd /opt ; unzip opentelemetry-webserver-sdk-x64-linux.tgz.zip; tar xvfz opentelemetry-webserver-sdk-x64-linux.tgz
RUN cd /opt/opentelemetry-webserver-sdk; ./install.sh
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/opentelemetry-webserver-sdk/sdk_lib/lib
RUN echo "load_module /opt/opentelemetry-webserver-sdk/WebServerModule/Nginx/1.23.1/ngx_http_opentelemetry_module.so;\n$(cat /etc/nginx/nginx.conf)" > /etc/nginx/nginx.conf
```

### 2. `docker-compose.yml`

使用 Docker Compose 來定義和運行多個服務，包括 Nginx、OpenTelemetry Collector、Prometheus 和 Grafana。

**內容說明：**

- **nginx 服務：**

  - **建構：** 使用 `Dockerfile.nginx` 建立自定義 Nginx 映像。
  - **掛載配置檔：** 將 `opentelemetry_module.conf` 掛載到容器的 `/etc/nginx/conf.d/opentelemetry_module.conf`。
  - **埠映射：** 將主機的 80 埠映射到容器的 80 埠。

- **otelcol（OpenTelemetry Collector）服務：**

  - **映像：** 使用最新的 `otel/opentelemetry-collector-contrib` 映像。
  - **掛載配置檔：** 將 `otel-collector-config.yml` 掛載到容器的 `/etc/otelcol-contrib/config.yaml`。
  - **掛載 Docker socket：** 掛載 `/var/run/docker.sock` 以收集 Docker 的指標資料。
  - **埠映射：** 開放 `4317` 和 `4318` 埠供接收資料。

- **prometheus 服務：**

  - **映像：** 使用 `prom/prometheus` 映像。
  - **掛載配置檔：** 將 `prometheus.yml` 掛載到容器的 `/etc/prometheus/prometheus.yml`。
  - **資料卷：** 使用 `prometheus-data` 卷來存儲數據。
  - **埠映射：** 將主機的 `9090` 埠映射到容器的 `9090` 埠。

- **grafana 服務：**

  - **映像：** 使用 `grafana/grafana` 映像。
  - **資料卷：** 使用 `grafana-storage` 卷來存儲 Grafana 的數據。
  - **埠映射：** 將主機的 `3000` 埠映射到容器的 `3000` 埠。
  - **環境變數：** 設定 Grafana 管理員密碼為 `admin`。
  - **依賴關係：** 依賴於 `prometheus` 服務，確保 Prometheus 在 Grafana 之前啟動。

- **資料卷定義：**

  - **prometheus-data：** 用於存儲 Prometheus 的數據。
  - **grafana-storage：** 用於存儲 Grafana 的數據。

```yaml
services:
    nginx:
        # https://opentelemetry.io/blog/2022/instrument-nginx/
        # https://docs.nginx.com/nginx/admin-guide/dynamic-modules/opentelemetry/
        build:
            dockerfile: Dockerfile.nginx
        volumes:
            - ./opentelemetry_module.conf:/etc/nginx/conf.d/opentelemetry_module.conf
        ports:
            - '80:80'

    otelcol:
        image: otel/opentelemetry-collector-contrib:latest
        volumes:
            - ./otel-collector-config.yml:/etc/otelcol-contrib/config.yaml
            - /var/run/docker.sock:/var/run/docker.sock # 使用 docker.sock 獲取 Docker 指標
        ports:
            - '4317:4317'
            - '4318:4318'

    prometheus:
        image: prom/prometheus
        volumes:
            - ./prometheus.yml:/etc/prometheus/prometheus.yml
            - prometheus-data:/prometheus
        ports:
            - '9090:9090'

    grafana:
        image: grafana/grafana
        volumes:
            - grafana-storage:/var/lib/grafana
        ports:
            - '3000:3000'
        environment:
            - GF_SECURITY_ADMIN_PASSWORD=admin
        depends_on:
            - prometheus

volumes:
    prometheus-data:
    grafana-storage:
```

### 3. `opentelemetry_module.conf`

Nginx 的 OpenTelemetry 模組配置文件，用於配置 OpenTelemetry 在 Nginx 中的行為。

**內容說明：**

- **啟用模組：** `NginxModuleEnabled ON;` 啟用 OpenTelemetry 模組。
- **設置 Span 導出器：** `NginxModuleOtelSpanExporter otlp;` 使用 OTLP 協議作為導出器。
- **設定導出器端點：** `NginxModuleOtelExporterEndpoint otelcol:4317;` 指定 OTLP 導出器的目標端點。
- **服務資訊：**

  - `NginxModuleServiceName DemoService;` 設定服務名稱為 `DemoService`。
  - `NginxModuleServiceNamespace DemoServiceNamespace;` 設定服務命名空間。
  - `NginxModuleServiceInstanceId DemoInstanceId;` 設定服務實例 ID。

- **其他配置：**

  - `NginxModuleResolveBackends ON;` 啟用後端解析。
  - `NginxModuleTraceAsError ON;` 將追蹤標記為錯誤。

```nginx
NginxModuleEnabled ON;
NginxModuleOtelSpanExporter otlp;
NginxModuleOtelExporterEndpoint otelcol:4317;
NginxModuleServiceName DemoService;
NginxModuleServiceNamespace DemoServiceNamespace;
NginxModuleServiceInstanceId DemoInstanceId;
NginxModuleResolveBackends ON;
NginxModuleTraceAsError ON;
```

### 4. `otel-collector-config.yml`

OpenTelemetry Collector 的配置文件，定義了如何接收、處理和導出數據。

**內容說明：**

- **接收器 (`receivers`)：**

  - **`otlp` 接收器：** 支持 gRPC 和 HTTP 協議，用於接收 OTLP 格式的追蹤和指標數據。
  - **`docker_stats` 接收器：** 通過 Docker Socket 獲取 Docker 容器的統計信息。

- **導出器 (`exporters`)：**

  - **`prometheus` 導出器：** 將收集到的指標數據導出，供 Prometheus 抓取。

- **服務配置 (`service`)：**

  - **管道 (`pipelines`)：** 定義了 `metrics` 管道，指定使用哪些接收器和導出器。
  - **遙測配置：** 設置收集器自身的指標，以便 Prometheus 進行監控。

```yaml
# https://opentelemetry.io/docs/collector/configuration/
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  docker_stats:
    # https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/dockerstatsreceiver
    endpoint: 'unix:///var/run/docker.sock' # 使用 Unix socket 路徑

exporters:
  prometheus:
    # https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/prometheusexporter
    endpoint: '0.0.0.0:8889'

service:
  pipelines:
    metrics:
      receivers: [otlp, docker_stats]
      exporters: [prometheus]
  telemetry:
    metrics:
      readers:
        - pull:
            exporter:
              prometheus:
                host: '0.0.0.0'
                port: 8888
```

### 5. `prometheus.yml`

Prometheus 的配置文件，用於指定要抓取的目標。

**內容說明：**

- **全域配置：**

  - **`scrape_interval`：** 設置全域抓取間隔為 5 秒。

- **抓取配置 (`scrape_configs`)：**

  - **`otel-collector` 抓取任務：** 指定 Prometheus 要抓取 OpenTelemetry Collector 的指標數據。

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'otel-collector'
    # metrics_path 默認為 '/metrics'
    # scheme 默認為 'http'。
    static_configs:
      - targets: ['otelcol:8888']
      - targets: ['otelcol:8889']
```

使用方法
-------

1. **克隆此倉庫：**

   ```bash
   git clone https://github.com/jim60105/docker-OpenTelemetry.git
   ```

2. **進入專案目錄：**

   ```bash
   cd docker-OpenTelemetry
   ```

3. **啟動 Docker 組合：**

   ```bash
   docker-compose up -d
   ```

4. **驗證服務運行：**

   - **Nginx：** 在瀏覽器中訪問 `http://localhost`，確認 Nginx 服務正常運行。
   - **Prometheus：** 在瀏覽器中訪問 `http://localhost:9090`，查看指標數據。
   - **Grafana：** 在瀏覽器中訪問 `http://localhost:3000`，使用用戶名 `admin` 和密碼 `admin` 登錄。

5. **配置 Grafana：**

   - 添加 Prometheus 作為資料來源，URL 設置為 `http://prometheus:9090`。
   - 導入相關的儀表板，以查看從 Nginx 和 Docker 收集的指標和追蹤數據。

參考資料
-------

- [在 Nginx 中使用 OpenTelemetry](https://opentelemetry.io/blog/2022/instrument-nginx/)
- [Nginx 的 OpenTelemetry 模組](https://docs.nginx.com/nginx/admin-guide/dynamic-modules/opentelemetry/)
- [.NET Core 中的可觀察性與 OpenTelemetry](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/observability-with-otel)
- [OpenTelemetry Collector 配置指南](https://opentelemetry.io/docs/collector/configuration/)
- [Docker Stats Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/dockerstatsreceiver)
- [Prometheus Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/prometheusexporter)

授權
-------

此專案以 MIT 授權條款發佈。詳情請參閱 [LICENSE](LICENSE)。
