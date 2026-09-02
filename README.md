# Robusta Deployment Guide — Kubernetes Pod Monitor Alerts

A practical guide to deploying [Robusta](https://robusta.dev/) on EKS (or any Kubernetes cluster) to monitor Pods and send targeted alerts to email and MS Teams channels.

Covers crash detection, OOM kills, liveness probe failures, and successful deployments — all routed using Pod labels defined in your Kubernetes deployment manifests (i.e. yaml files).

---

## Overview

This guide walks through deploying Robusta as a lightweight Kubernetes alerting layer — **without** Prometheus or a full observability stack. The focus is narrow and practical: get the right email and MS Teams alerts to the right team when something goes wrong with their pods.

**What you get:**

- Email alerts for `CrashLoopBackOff`, `OOMKilled`, liveness probe failures, deployment updates and deployment creations
- Per-team routing via pod labels (i.e., `notify: "devops"`) — no shared inboxes
- Gmail or company SMTP integration with app-password authentication
- Secrets managed via Kubernetes secrets (not hardcoded in values)
- Resource-bounded Robusta runner and forwarder pods

**What this guide does NOT cover:**
- Prometheus, Grafana, or metrics-based alerting
- Slack, PagerDuty, or other notification sinks
- Multi-cluster setups

---

## Motivation

Most Kubernetes alerting setups require a full Prometheus stack — Prometheus, Alertmanager, and Grafana — just to get notified when a Pod crashes. That's significant overhead if you only need email or MS Team alerts for DevOps team.

Robusta offers a lighter alternative: it watches the Kubernetes API event stream directly and fires playbooks based on pod lifecycle events. No metrics pipeline required.

Basically whenever one of the following Kubernetes events is detected, alerts will be generated to both mailing list(s) and MS Team channel(s):

1. `OOMKilled`: container killed for exceeding its memory limit → Kubernetes restarts the container
2. `CrashLoopBackOff`: container stuck in a repeated crash-and-restart loop → Kubernetes   restarts the container repeatedly, with increasing backoff delay each time
3. `on_kubernetes_warning_event_create`: Liveness probe prob failure → probe fails → Kubernetes restarts the container
4. `on_deployment_update`:  Deployment update → any Kubernetes yaml change under `spec` → Pod (and its containers) is replaced, not restarted. A new container image version was deployed
5. `on_deployment_create:` A Deployment is created for the first time for a given project

Note 'container restart' happens in events 1, 2 and 3 above so let's be precise about what it actually does. Basically the same Pod, same node, same Pod IP but the container process inside it gets killed and started again.

Such alerts are important to notify the relevant team in real time whenever a deployment changes, or whenever a container starts crashing or failing health checks.

The specific problem this guide will also solve: in a shared cluster, different teams own different deployments. A single shared alert inbox creates noise and unclear ownership. By labelling pods with `notify: <team>` and scoping each email sink to its label in Kubernetes yaml file, each team only receives alerts for pods they own.

---
## Prerequisites

Before deploying, ensure you have:

- `kubectl` configured and pointed at your target cluster
- `helm` v3 installed
- A Gmail account with an [App Password](https://support.google.com/accounts/answer/185833) generated (2FA must be enabled)
- Namespace-level permissions to create secrets and install Helm releases

---
## Deploy Robusta

### 1. Add the Helm repository

```bash
helm repo add robusta https://robusta-charts.storage.googleapis.com
helm repo update
```

This registers Robusta's chart repository and fetches the latest index. You only need to do this once per machine.

### 2. Create the email secret

Since we don't want to hardcode SMTP credentials in Robusta values file **robusta-values.yaml**, logic suggests using Kubernetes secrets to inject it into Robusta values file like

```bash
kubectl create secret generic email-secret \
  --namespace robusta \
  --from-literal=EMAIL_USER='your@gmail.com' \
  --from-literal=EMAIL_PASSWORD='your-app-password'
```

then references it in robusta-values.yaml like
```yaml
runner:
  additional_env_vars:
    - name: EMAIL_USER
      valueFrom:
        secretKeyRef:
          name: email-secret
          key: EMAIL_USER
    - name: EMAIL_PASSWORD
      valueFrom:
        secretKeyRef:
          name: email-secret
          key: EMAIL_PASSWORD
```

But unfortunately, the current version of Robusta version doesn't support this (at the time of writing this) so a workaround is required. Before going any further, let's recap the required SMTP config for sending email alert:

* Gmail format:
	mailtos://your-mail.com:APP-PASSWORD@gmail.com?to=recipient@company.com

* Corporate SMTP format:
	mailtos://user:password@your.smtp.server:465?from=alerts@yourcompany.com&to=recipient@company.com

Now since it's not possible to use Kubernetes secrets to do something like below in robusta-values.yaml
```bash
mailtos://${EMAIL_USER}:${EMAIL_PASSWORD}@...
```
where both ${EMAIL_USER} and ${EMAIL_PASSWORD} get replaced with Kubernetes secrets, the next best thing is to create Kubernetes secret representing the entire config line then inject it into robusta-values.yaml during deployment.

To create actual K8s email secret that can be injected into robusta-values.yaml during deployment, we do something like
```bash
# Replace ${user} and ${password} below with actual SMTP user and password before running the command.
# 
# Run once to create the secret. To update it, delete and recreate:
#   kubectl delete secret email-secret -n robusta 
kubectl create secret generic email-secret \
  --namespace=robusta \
  --from-literal=MAILTO_URL="mailtos://${user}:${password}@company-domain.com:587?to=devops@company-domain.com" 

# For each deployment, run this 1st to pull the secret value of SMTP credential out of Kubernetes
MAILTO=$(kubectl get secret email-secret -n robusta \
  -o jsonpath='{.data.MAILTO_URL}' | base64 -d)
```

### 3. Actual Deployment

```bash
# Run this to deploy Robusta with SMTP credential injection. Note this command can also be used to restart Robusta.
helm upgrade --install robusta robusta/robusta \
  --version 0.42.0 \
  -f robusta-values.yaml \
  --set "sinksConfig[0].mail_sink.mailto=${MAILTO}" \
  -n robusta \
  --create-namespace
 
# Check the 2 Robusta components, runner and forwarder are up and running
kubectl get pods -n robasta

# Force restart of Robusta's runner & forwarder components if 'kubectl get pods -n robusta' shows
# both runner and forwarder components are not using the latest, i.e. age of runner is hrs not seconds.
kubectl rollout restart deployment/robusta-runner -n robusta
kubectl rollout restart deployment/robusta-forwarder -n robusta

# Check runner log for error if no alert is received after deployment
kubectl logs -n robusta -l app=robusta-runner  -f --tail=50
```


Note Robusta deploys two pods:

| Pod                 | Role                                                                      |
| ------------------- | ------------------------------------------------------------------------- |
| `robusta-runner`    | Executes playbooks and sends notifications                                |
| `robusta-forwarder` | Watches Kubernetes API event stream and forwards events to robusta-runner |

Verify both are running:

```bash
kubectl get pods -n robusta
```

> `helm upgrade` applies a rolling update to the runner pod. Changes take effect once the pod restarts.

---

## Configuration Explained

Let's walk through the key settings in the supplied `robusta-values.yaml` to ensure everything is crystal clear.
### Disabling the Prometheus stack

```yaml
enablePrometheusStack: false
enablePlatformPlaybooks: false
builtinPlaybooks: []
```

Robusta ships with optional Prometheus/Grafana integration. For email-only alerting, these are disabled entirely to reduce resource usage and complexity.

### Resource limits

```yaml
runner:
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "200m"

forwarder:
  resources:
    requests:
      memory: "64Mi"
      cpu: "50m"
    limits:
      memory: "128Mi"
      cpu: "200m"
```

The forwarder CPU limit is especially important — the Helm chart does not set one by default, and an unbounded forwarder can starve other pods on the same node.

# Email sinks — per-team routing

Each team gets a dedicated `mail_sink`. The `scope.include` filter ensures the sink only accepts alerts from pods with the matching label.

```yaml
sinksConfig:
  - mail_sink:
      name: email-devops-team
      mailto: "mailtos://USER:APP-PASSWORD@gmail.com?to=manager@company.com"
      with_header: true
      default: false
      scope:
        include:
          - labels: "notify=devops"
```

Key fields:

| Field            | Purpose                                                        |
| ---------------- | -------------------------------------------------------------- |
| `name`           | Referenced by playbooks to route alerts                        |
| `mailto`         | SMTP URL — see format notes below                              |
| `default: false` | Sink only receives alerts explicitly sent to it                |
| `scope.include`  | Pod label filter — alerts from other pods are silently dropped |

**To add a new team:**

1. Duplicate the `mail_sink` block
2. Change `name`, `mailto` (recipient), and `scope.include` label value
3. Add `notify: <new-team>` to the target deployment's pod template labels
4. Add the new sink name to each playbook's `sinks` list

# MS Team sinks — per-team routing

Robusta is capable of sending alerts to Microsoft Team chatroom or channel as well. 
Complete configuration is like
```bash
 # ── MS Teams ────────────────────────────────────────────────────────
  # Sends alerts to a shared Teams channel via Incoming Webhook.
  # scope.include ensures only labeled pods can trigger it.
  #
  # How to get the webhook URL:
  #   1. In Teams, open the channel you want alerts in
  #   2. Click ··· → Connectors → "Incoming Webhook" → Configure
  #   3. Name it (e.g. "Robusta K8s Alerts") and click Create
  #   4. Copy the URL and paste it below
  #
  # For newer Teams Workflows webhook (recommended by Microsoft):
  #   Channel → Workflows → "Post to a channel when a webhook request is received"

  # Use this sink to send alert to MS Teams 'DevOps' Chat
  - ms_teams_sink:
      name: ms-teams-alerts-devops
      webhook_url: "REPLACE_WITH_WEBHOOK_URL"
      default: true   # required — Robusta needs at least one default sink or all alerts are dropped
      scope:
       include:
         - labels: "notify=devops"
"
```

The steps to get value of webhook_url is like
1. Click '...' in left hand menu item => click 'Search in Store'

   <a href="images/api-service.jpg">
   <img src="images/ms-team-webhook-1.png" width="450" alt="Click to enlarge">
   </a>

2. Search webhook, choose 'Send webhook alerts to a chat' then follow the process from there to generate a webhook URL for a chosen chat

   <a href="images/api-service.jpg">
   <img src="images/ms-team-webhook-2.png" width="450" alt="Click to enlarge">
   </a>

Note the webhook URL to a chat is to be treated as a secret, don't save it to git as anyone who has access to this URL can send messages to the chat.

### Playbooks — alert triggers

Each playbook defines a trigger, enrichment actions, and which sinks receive the alert.

#### CrashLoopBackOff

```yaml
- name: "CrashLoop Alert"
  triggers:
    - on_pod_crash_loop:
        restart_count: 1
        restart_reason: "Error"
  actions:
    - create_finding:
        title: "Pod $name crashed in namespace $namespace"
        severity: HIGH
        repeat_delay: 120
    - logs_enricher:
        lines: 100
    - pod_events_enricher: {}
  sinks:
    - email-infra-team
    - ms-teams-alerts-devops
```

- Fires after the first restart (not zero) to avoid false positives during normal pod startup
- `repeat_delay: 120` prevents alert storms during persistent crash loops — one email per 2 minutes maximum
- `logs_enricher` attaches the last 100 log lines to the email
- `pod_events_enricher` attaches the pod's Kubernetes event history

#### OOMKilled

```yaml
- name: "OOMKill Alert"
  triggers:
    - on_pod_oom_killed: {}
  actions:
    - create_finding:
        title: "Pod $name was OOMKilled in namespace $namespace"
        severity: HIGH
        repeat_delay: 120
    - logs_enricher:
        lines: 100
  sinks:
    - email-infra-team
    - ms-teams-alerts-devops
```

Triggered when the Linux kernel OOM killer terminates a container due to memory pressure.

#### Liveness Probe Failure

```yaml
- name: "Liveness Probe Fail Alert"
  triggers:
    - on_kubernetes_warning_event_create:
        include: ["Liveness"]
  actions:
    - create_finding:
        title: "Pod $name restarted — liveness probe failed in namespace $namespace"
        severity: HIGH
        repeat_delay: 120
    - event_resource_events: {}
  sinks:
    - email-infra-team
    - ms-teams-alerts-devops
```

Listens for Kubernetes warning events containing "Liveness" — emitted when a container's liveness probe exceeds its `failureThreshold`.

#### Deployment Update

```yaml
  # Deployment Update Alert triggered by 'kubectl apply -f ...' or 'kubectl rollout restart ...'. This will only be
  # triggered if there is actual YAML change.
  - name: "Deployment Update Alert"
    triggers:
      - on_deployment_update:
          change_filters:
            # 1. Ignore the status noise entirely
            ignore:
              - status
              - metadata.resourceVersion
              - metadata.managedFields
            # 2. Only fire if the underlying spec configuration actually changed
            include:
              - spec
    actions:
      # Generate the clean diff comparison in alert message
      - resource_babysitter: {}

      # Intercepts the diff alert and rewrites its title using Robusta tokens
      - customise_finding:
          title: "Deployment $name updated in namespace $namespace"
    sinks:
      - email-devops-team
      - ms-teams-alerts-devops
```

Fires event when a deployment's `spec` changes (new image, environment variables, replica count, etc.). The `ignore` list filters out high-frequency noise fields that change constantly without a human-initiated change.

#### Deployment Creation
```bash
  # Alert when a Deployment is first created (not on subsequent updates) for a given project
  - name: "Deployment Created Alert"
    triggers:
      - on_deployment_create: { }
    actions:
      - create_finding:
          title: "Deployment $name created in namespace $namespace"
          severity: "INFO"
          aggregation_key: "DeploymentCreated"
    sinks:
      - email-devops-team
      - ms-teams-alerts-devops
```

Note each playbook has specified 2 sinks, leading to alert in both email and MS Team chatroom

---

## Opting a Deployment into Alerts

To join a Kubernetes deployment into Robusta Pod monitor is very simple, just add two `notify` labels to deployment yaml file as below. A complete sample yaml file `microservices-bootstrap.yaml` will be provided in this guide for reader.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: microservices-bootstrap
  namespace: default
  labels:
    notify: "devops" // Label on the DEPLOYMENT object to detect deployment events by Robusta

spec:
  replicas: 1
  selector:
    matchLabels:
      app: microservices-bootstrap
  template:
    metadata:
      labels:
        app: microservices-bootstrap
        notify: "devops" // Label on the Pod template to detect Pod events by Robusta
  ...      
```

Notice the value of metadata.labels.notify and spec.template.metadata.labels.notify is "devops", matching value of both Email Sink and MS Team Sink config above, e.g.

```yaml
- ms_teams_sink:
      ...
      scope:
       include:
         - labels: "notify=devops"
```

Pods without a `notify` label are ignored by all sinks from Robusta. 

---

## Trigger Reference

| Trigger                | When it fires                                                                            |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| `CrashLoopBackOff`     | Container exits with error; Kubernetes restarts it                                       |
| `OOMKilled`            | Container killed by Linux kernel OOM; Kubernetes restarts it                             |
| Liveness probe failure | `failureThreshold` breached; kubelet kills and restarts container                        |
| Deployment Update      | Any change to`spec` within K8s yaml will fires once at deployment level, not per replica |
| Deployment Creation    | Fires once when deployment is created for the first time                                 |

---

## Testing

Let's see Robusta in action with some examples.

Deploy a project for the first time in console with Kubernetes command like
* `kubectl apply -f microservices-bootstrap.yaml -n default`

will generate an Email alert like

  <a href="images/api-service.jpg">
  <img src="images/deployment-creation.png" width="600" alt="Click to enlarge">
  </a>

Rolling out new Pod with Kubernetes command like
* `kubectl rollout restart deployment/{deployment-name} -n {namespace}`

will generate MS Team alert like

  <a href="images/api-service.jpg">
  <img src="images/deployment-update.png" width="600" alt="Click to enlarge">
  </a>	

---

## Resources

- [Robusta documentation](https://docs.robusta.dev/)
- [Robusta playbook reference](https://docs.robusta.dev/master/playbook-reference/)
- [Gmail App Passwords](https://support.google.com/accounts/answer/185833)
- [apprise URL formats (for mailto)](https://github.com/caronc/apprise/wiki/Notify_email)

---
Author: Samuel Huang
