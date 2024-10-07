
# Minikube Logging with Loki and Grafana

## 1. Create an Audit Policy

Create an audit policy in a path that is mounted to your Minikube cluster:

```bash
mkdir -p ~/.minikube/files/etc/ssl/certs

cat <<EOF > ~/.minikube/files/etc/ssl/certs/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  verbs: ["create", "delete"]
  resources:
  - group: ""
    resources: ["pods", "namespaces"]
EOF
```

## 2. Start Minikube

Start your Minikube cluster with the following configuration:

```bash
minikube start --driver=docker --nodes=2 --memory=2048 --cpus=2 \
--extra-config=apiserver.audit-policy-file=/etc/ssl/certs/audit-policy.yaml \
--extra-config=apiserver.audit-log-path=-
```

## 3. Enable Ingress

Enable the Ingress addon in Minikube:

```bash
minikube addons enable ingress
```

## 4. Install Grafana and Loki

Add the Grafana Helm repository and install Loki Stack:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### Loki Helm values (`loki-stack-values.yaml`)

```yaml
loki:
  enabled: true
  size: 1Gi
promtail:
  enabled: true
grafana:
  enabled: true
  sidecar:
    datasources:
      enabled: true
grafana:
  enabled: true
  image:
    tag: "10.2.5"
```

Install Loki Stack:

```bash
helm upgrade --install loki --namespace=loki-stack grafana/loki-stack \
--values loki-stack-values.yaml --create-namespace
```

Update Loki to version 2.9.3:

```bash
kubectl edit StatefulSet/loki -n loki-stack
```

## 5. Configure Grafana Agent

Create Helm values for Grafana Agent:

```yaml
agent:
  mode: 'flow'
  configMap:
    create: true
    content: |
      logging {
        level  = "info"
        format = "logfmt"
      }
      loki.source.kubernetes_events "events" {
        log_format = "json"
        forward_to = [loki.write.loki_endpoint.receiver]
      }
      loki.write "loki_endpoint" {
        endpoint {
          url = "http://loki.loki-stack:3100/loki/api/v1/push"
        }
      }
```

Install Grafana Agent:

```bash
helm upgrade --install grafana-agent --namespace=loki-stack grafana/grafana-agent \
--values grafana-agent-values.yaml
```

## 6. Access Grafana Dashboard

Forward the Grafana service port:

```bash
kubectl port-forward svc/loki-grafana 3000:80 -n loki-stack
```

Retrieve the Grafana admin password:

```bash
kubectl get secret loki-grafana -n loki-stack -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

## 7. Example LogQL Query

Here's an example query for tracking `create` and `delete` events for namespaces and pods:

```logql
{component="kube-apiserver"} | json | line_format "{{.log}}" | json \
| (verb="create" or verb="delete") \
| annotations_authorization_k8s_io_decision!="" \
|~ ".*\"resource\":\"(namespaces|pods)\".*"
```

## 8. Configure SMTP for Grafana

Create a ConfigMap for SMTP configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-smtp-config
  namespace: default
data:
  grafana.ini: |
    [smtp]
    enabled = true
    host = smtp.gmail.com:587
    user = your-email@gmail.com
    password = your-app-password  # Use the App Password here
    from_address = your-email@gmail.com
    from_name = Grafana
    startTLS_policy = "Yes"
```

## 9. Set up Query Alert in Logs

You can now set up alerting in Grafana based on the log queries.

```
count_over_time({component="kube-apiserver"} 
| json 
| line_format "{{.log}}" 
| json 
| (verb="create" or verb="delete") 
| annotations_authorization_k8s_io_decision!="" 
|~ ".*\"resource\":\"(namespaces|pods)\".*" 
[1m])
```

*Edit Alert Rule page in Grafana step by step*

1. Define Query and Alert Condition
Query: This is where you define the log query you want to use to trigger the alert. You use the LogQL syntax for querying logs from Loki. In your case, the query checks for log entries related to the creation or deletion of pods or namespaces.

[5m]: This is the time range over which Grafana evaluates the log data. It specifies that Grafana will consider log entries in the last 5 minutes when evaluating the alert condition. The alert will be evaluated at the end of this time window. If your interval is set to [1m], Grafana will check for log entries every minute, using the last minute's data.

count_over_time(...): This function counts the number of log entries that match the query over the specified time range. For example, count_over_time(query[5m]) counts how many logs matched your query in the last 5 minutes.

Other options you can use include:
sum_over_time(...): Sums numerical values in logs over time.
avg_over_time(...): Averages numerical values over the specified time range.
min_over_time(...) / max_over_time(...): Finds the minimum or maximum value over time.
Expressions Tab
The Expressions tab is where you can create additional calculations based on the output of the query defined above.

B Reduce: This option allows you to manipulate the output of the query. For example, you can reduce the results to a single value using functions like count, sum, avg, etc.

Function: The function you choose determines how the output will be processed. For example, if you choose count, it will return the total count of logs matching the query within the defined range.

Count: This refers to the number of occurrences that match your query within the specified time range.

Modes:

Strict: Will only alert if the result strictly exceeds the threshold.
Not Strict: Allows for some flexibility and may trigger if conditions are close to the threshold.
Threshold Tab
This is where you define the conditions that will trigger the alert based on the output from the previous steps.

You can set the threshold to trigger the alert based on the results of the Reduce function. For example, setting it to > 1 means that if the count of matching logs is greater than 1, the alert will trigger.

2. Set Evaluation Behavior
Evaluation Group: This is a label for grouping alerts that share the same evaluation criteria. Grouping alerts helps manage notifications and reduces noise.

Pending Period: This is the duration that the alert must remain in the pending state before it is considered to have triggered. If the alert condition is met for this period, the alert will change from pending to firing. This helps to prevent false positives from brief fluctuations in log counts.

3. Add Annotations
Annotations are additional metadata added to the alert for better context. They can help provide more information when the alert is triggered. For example, you can add a description of the alert or any specific notes that will help in understanding why it was triggered.
4. Labels and Notifications
Labels: Labels are key-value pairs that can be used to identify or categorize alerts. For instance, you could label an alert with severity=critical to signify its importance.

Notifications: This section is where you configure how and where to send notifications when the alert triggers. You can set up various notification channels (e.g., email, Slack) and specify which alerts should notify which channels.