# Helm bug report

Helm version: `version.BuildInfo{Version:"v3.11.2", GitCommit:"912ebc1cd10d38d340f048efaf0abda047c3468e", GitTreeState:"clean", GoVersion:"go1.18.10"}`

I noticed some odd behavior of helm when a chart has 2 different sub-charts, each of them with a dependency on the same chart. I have created a minimal example to reproduce the issue.

## Steps to reproduce

Clone the repo: `https://github.com/yardenshoham/helm-bug-report.git` and run `helm template reproduce .`. You should see the following output:

```
coalesce.go:223: warning: destination for postgresql.ldap.tls is a table. Ignoring non-table value ()
Error: template: reproduce/charts/keycloak/charts/postgresql/templates/primary/statefulset.yaml:433:20: executing "reproduce/charts/keycloak/charts/postgresql/templates/primary/statefulset.yaml" at <include "postgresql.readinessProbeCommand" .>: error calling include: template: reproduce/charts/airflow/charts/postgresql/templates/_helpers.tpl:120:15: executing "postgresql.port" at <.Values.service.port>: nil pointer evaluating interface {}.port

Use --debug flag to render out invalid YAML
```

You can also run `helm create reproduce` and add the following to `reproduce/Chart.yaml`:

```
dependencies:
  - name: airflow
    version: 10.3.1
    repository: https://github.com/bitnami/charts/raw/archive-full-index/bitnami
  - name: keycloak
    version: 13.3.0
    repository: https://github.com/bitnami/charts/raw/archive-full-index/bitnami
```

Then run `helm dependency build reproduce` and `helm template reproduce reproduce` and you should see the same error.

## What I think is happening

Both the airflow chart and the keycloak chart have a dependency on the postgresql chart. Airflow has a dependency on the postgresql chart version 10.9.3 and keycloak has a dependency on the postgresql chart version 12.2.1. When helm is trying to render the keycloak chart, it is using the postgresql chart version 10.9.3 `_helpers.tpl` instead of the version 12.2.1. This is because the postgresql chart version 10.9.3 is the first postgresql chart that is found in the `charts` directory of the keycloak chart. So there is a bug in the way helm is looking for the postgresql chart version 12.2.1.

Because both charts use functions with the same name they override each other.

## Expected behavior

There should be isolation between the charts. Each chart should use its own dependencies and not use the dependencies of other charts. I see no reason for the current behavior. I should be able to use any version of a chart as a dependency without worrying about other charts using the same chart.
