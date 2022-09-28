# Environment Setup
This is a simple demo walkthrough to setup basic node.js application and instrument it with OTEL and sending it to Grafana Agent and Grafana Cloud.

## Basic application `app.js`
This is a simple node.js application for this demo.
```js
/* app.js */

const express = require("express");

const PORT = process.env.PORT || "8080";
const app = express();

app.get("/", (req, res) => {
  res.send("Hello World");
});

app.listen(parseInt(PORT, 10), () => {
  console.log(`Listening for requests on http://localhost:${PORT}`);
});
```



## Step 1: Install Core Depedencies
GOAL: Install tracing SDK dependencies
```bash
npm install @opentelemetry/sdk-node @opentelemetry/api
```

## Step 2: Install Exporter
GOAL: OTLP Exporter
This scenario chose OTLP HTTP.
```bash
npm install --save @opentelemetry/exporter-trace-otlp-http
```

## Step 3: Setup Auto Instrumentations Module
GOAL: Open Telemetry Auto Instrumentation Module. More modules in [registry](https://opentelemetry.io/registry/?language=js&component=instrumentation).
```bash
npm install @opentelemetry/auto-instrumentations-node
```

## Step 4: Setup Tracing `tracing-exporter.js`
GOAL: This holds all your tracing setup code.
IMPORTANT: The tracing setup and configuration should be run before your application code!
```js
// tracing-exporter.js

'use strict'

const process = require('process');
const opentelemetry = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
// const { ConsoleSpanExporter } = require('@opentelemetry/sdk-trace-base');
const {
  OTLPTraceExporter,
} = require("@opentelemetry/exporter-trace-otlp-http");
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

// configure the SDK to export telemetry data to the console
// enable all auto-instrumentations from the meta package
const traceExporter = new OTLPTraceExporter({
    // optional - url default value is http://localhost:4318/v1/traces
    url: "<your-otlp-endpoint>/v1/traces",
    // optional - collection of custom headers to be sent with each request, empty by default
    headers: {},
  });
const sdk = new opentelemetry.NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'my-service',
  }),
  traceExporter,
  instrumentations: [getNodeAutoInstrumentations()]
});

// initialize the SDK and register with the OpenTelemetry API
// this enables the API to record telemetry
sdk.start()
  .then(() => console.log('Tracing initialized'))
  .catch((error) => console.log('Error initializing tracing', error));

// gracefully shut down the SDK on process exit
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Tracing terminated'))
    .catch((error) => console.log('Error terminating tracing', error))
    .finally(() => process.exit(0));
});
```

## Step 5: Grafana Agent Setup
> [!TIP] Do this before Setup Tracing !

- Run Grafana Agent Container with exposed port 4317 and 4318 for OTLP
- Configure agent.yaml with automatic_logging (optional)
- replace `/path/to/config.yaml` with the directory of your agent.yaml
```bash
docker run -d -p 4317:4317 -p 4318:4318 -v /tmp/agent:/etc/agent/data -v /path/to/config.yaml:/etc/agent/agent.yaml grafana/agent:v0.27.0
```

```yaml
# agent.yaml
traces:
  configs:
  - name: default
    remote_write:
      - endpoint: <YOUR-GRAFANACLOUD-URL>
        basic_auth:
          username: <YOUR-USERNAME>
          password: <YOUR-API-KEY>
    receivers:
      jaeger:
        protocols:
          grpc:
          thrift_binary:
          thrift_compact:
          thrift_http:
      zipkin:
      otlp:
        protocols:
          http:
          grpc:
      opencensus:
    automatic_logging:
	  backend: logs_instance
	  logs_instance_name: default
	  roots: true
      spans: true
      processes: true
      overrides:
      #override traceID key for logs and traces correlation in Grafana Cloud.
        trace_id_key: 'traceID'

logs:
  configs:
  - name: default
    clients:
    - url: <YOUR-GRAFANACLOUD-URL>
      basic_auth:
        username: <YOUR-USERNAME>
        password: <YOUR-API-KEY>
      external_labels:
        job: 'my-service-autologs'
    positions:
          filename: /tmp/positions.yaml
```

## Step 6: Run Application with Tracing Code
GOAL: Run application with the --require flag to load the tracing code before the application code.
```bash
node --require './tracing-exporter.js' app.js
```

## Step 7: Test your tracing
Access `localhost:8080` and refresh for a few times.
GO to Grafana Cloud and look for your traces.

![screnshot1](https://github.com/alfredo-d/demo-otel-js/blob/main/attachments/OTEL%20Instrumentation%20Node.js_Grafana%20Cloud%20Trace%20Sample.png)

![screenshot2](https://github.com/alfredo-d/demo-otel-js/blob/main/attachments/OTEL%20Instrumentation%20Node.js_Grafana%20Cloud%20Trace%20Sample%20with%20Automatic%20Logging.png)

---
Reference:
[OTEL node.js instrumentation](https://opentelemetry.io/docs/instrumentation/js/getting-started/nodejs/)
