# Equinix Helm chart for the OpenTelemetry Collector

Originally based on [Honeycomb's Helm chart](https://github.com/honeycombio/helm-charts/tree/main/charts/opentelemetry-collector).

This chart does the following:

- creates a Kubernetes [deployment](templates/deployment.yaml) of pods that use the specified version of the
  [otel/opentelemetry-collector-contrib Docker image](https://hub.docker.com/r/otel/opentelemetry-collector-contrib) from Docker Hub
- mounts a volume for the [Collector configuration file](templates/opentelemetry-collector-config.yaml), which is generated from a template that pulls cluster metadata from Atlas
- creates a Kubernetes [service](templates/service.yaml) for the Collector, exposing relevant ports
- pulls the Honeycomb secret in keymaker for the application namespace the Collector is deployed to

That last step is specific to the deployment strategy we're using at Metal:
**one Collector "app" per application namespace** (e.g. cacher, narhwal, boots, etc.).

## Configuring your application to use the Collector

Before deploying the Collector itself to your application namespace, you will need to update your app configuration to enable sending telemetry data to the Collector.

### Add OpenTelemetry instrumentation to the application code

Go apps should use [equinix-labs/otel-init-go](https://github.com/equinix-labs/otel-init-go).
Follow the configuration instructions in the README.

For Ruby apps, follow the instructions in [Confluence](https://packet.atlassian.net/l/c/XBP11Ef4).

### Set the OTLP endpoint in the app Helm chart

Your application's OpenTelemetry SDK configuration looks for the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable to determine where to send data.
In this case the OTLP endpoint is the url of the collector deployed to that application's namespace.

For apps sending OTLP over HTTP (Ruby apps), use the HTTP endpoint:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT="http://opentelemetry-collector:55681"
export OTEL_RUBY_EXPORTER_OTLP_SSL_VERIFY_NONE=true
```

For apps sending OTLP over gRPC (Go apps), use the gRPC endpoint:

```shell
export OTEL_EXPORTER_OTLP_ENDPOINT="opentelemetry-collector:4317"
export OTEL_EXPORTER_OTLP_INSECURE=true
```

Depending on the app's Helm chart configuration, the environment variable may need to be set in different ways.
Most k8s-site-{appname} charts will set environment variables in `values.yaml` like so:

```yaml
{appname}:
  env:
    . . .
    OTEL_EXPORTER_OTLP_ENDPOINT: "opentelemetry-collector:4317"
    OTEL_EXPORTER_OTLP_INSECURE: "true"
```

If you're not sure where to add the environment variable, ask SRE (`#sre`) or the Delivery team (`#eng-k8s`) for help.

### Generate a Honeycomb secret for your app and add it to keymaker

Generate a new API key in Honeycomb following [these instructions in Confluence](https://packet.atlassian.net/wiki/spaces/SWE/pages/2794946582/Deploying+the+OpenTelemetry+Collector#Set-up-Honeycomb-secrets).
The API key name in Honeycomb should use the format `prod-{appname}`.

Push the key to keymaker following [these instructions on the delivery docs site](https://delivery-docs.metalkube.net/core_services/keymaker/?h=keymaker#add-secret-to-secret-store). A typical push will look like below. Replace the all caps values, the rest should be consistent across apps.

```yaml
apiVersion: keymaker.equinixmetal.com/v1
kind: ExternalSecretPush
metadata:
  name: honeycomb-secret
spec:
  backend: ssm
  environment: prod
  namespace: REPLACE_ME_NAMESPACE
  secrets:
    - key:   honeycomb-secret
      value: REPLACE_ME_HONEYCOMB_TOKEN
      version: v1
```

The final key path should look like `/prod/{appname}/honeycomb-secret/v1`, which will automatically get picked up by the ExternalSecretPull generated by the template in this chart.

### Syncing in Argo

For initial initial deployment and any changes to the OTLP endpoint, the app's pods will need to be restarted in order to pick up the new/updated environment variables.
For some configurations, Argo will restart the pods automatically.
For others, you may need to update the Helm chart so that Argo detects a change in the environment variables.
Reach out to SRE (`#sre`) or the Delivery team (`#eng-k8s`) for help with that.

## Deploying the collector

### Add the Collector chart to your app's namespace in Atlas

In the `packethost/delivery-infrastructure` repository, edit `atlas/apps.yaml`.
Add `otel-collector` and the repoURL (git URL) as an additional list item under the `apps` section of your application's configuration:

```yaml
    - name: otel-collector
      repoURL: "git@github.com:packethost/k8s-otel-collector.git"
```

For a better idea of where it goes, see
[the diff for cacher](https://github.com/packethost/delivery-infrastructure/pull/1773/commits/06a8b3f9c8ddd37b7068d38f6aef055e985a09b9).

## Troubleshooting

[Follow the instructions in these docs](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/troubleshooting.md)
to set the Collector's own logs to `DEBUG`.

## Architecture

By default, we're deploying one collector instance per application namespace.
