# 🤖 OpenShift Self-Healing with AI Root Cause Analysis

This project demonstrates an **AI-powered incident detection, Root Cause Analysis (RCA), and remediation workflow for Red Hat OpenShift**.
<img width="1536" height="1024" alt="OpenShift AI Self-Healing Lab  Prometheus_ AAP EDA   AI Root Cause Analysis" src="https://github.com/user-attachments/assets/b1f739f0-4d14-4d51-88aa-7e3db53a16bc" />

The solution combines:

- Red Hat OpenShift
- Prometheus
- Alertmanager
- Red Hat Ansible Automation Platform (AAP)
- Event-Driven Ansible (EDA)
- AAP Workflow Templates
- FastAPI
- OpenAI
- Kubernetes NetworkPolicy

Instead of simply restarting a failed workload, the automation collects OpenShift diagnostic evidence, sends it to an AI RCA service, identifies the probable root cause, and recommends an appropriate remediation.

> **Part 2B:** AI-Powered Root Cause Analysis and Automated Remediation

---

# 🏗️ Architecture

```text
OpenShift Application
        │
        ▼
Prometheus
        │
        ▼
PrometheusRule
        │
        ▼
Alertmanager
        │
        ▼
AAP External Event Stream
        │
        ▼
Event-Driven Ansible
        │
        ▼
EDA Rulebook
        │
        ▼
AAP Workflow Template
        │
        ├── Collect OpenShift Diagnostics
        │
        ▼
AI RCA Service
FastAPI + OpenAI
        │
        ▼
Structured RCA
        │
        ├── Root Cause
        ├── Confidence Score
        ├── Recommended Action
        └── Remediation Parameters
        │
        ▼
Ansible Remediation
        │
        ├── ServiceAccount / SCC Fix
        └── NetworkPolicy Fix
        │
        ▼
✅ Application Restored
```

---

# 🎯 Lab Scenarios

Two different OpenShift failure scenarios are demonstrated.

## Scenario 1 – ServiceAccount / SCC Issue

An NGINX workload fails because the application does not have the required OpenShift Security Context Constraint permissions.

```text
NGINX
   ↓
Pod Failure
   ↓
Prometheus Alert
   ↓
EDA
   ↓
Collect Diagnostics
   ↓
AI RCA
   ↓
ServiceAccount / SCC identified
   ↓
Ansible Remediation
   ↓
Application Running
```

Example SCC command used during the lab:

```bash
oc adm policy add-scc-to-user anyuid -z nginx-sa -n demo
```

Verify:

```bash
oc get pod -n demo

oc describe pod <pod-name> -n demo
```

---

# 🌐 Scenario 2 – NetworkPolicy / MySQL Connectivity

The second scenario demonstrates a more complex application dependency.

Two namespaces are used:

```text
frontend
backend
```

The frontend application attempts to connect to:

```text
mysql-service.backend.svc.cluster.local:3306
```

A NetworkPolicy prevents the connection.

Application logs show:

```text
❌ Unable to connect to mysql-service.backend.svc.cluster.local:3306
Error: timed out
```

The AI analyzes OpenShift diagnostic information and determines that network isolation is the probable root cause.

It then provides the parameters needed to create a least-privilege NetworkPolicy.

---

# 🚀 Create the Lab Namespaces

```bash
oc new-project frontend

oc new-project backend
```

Verify:

```bash
oc get projects
```

---

# 🗄️ Deploy MySQL

Deploy MySQL in the `backend` namespace:

```bash
oc create deployment mysql \
  --image=mysql \
  -n backend
```

Configure the root password:

```bash
oc set env deployment/mysql \
  MYSQL_ROOT_PASSWORD=MyStrongPassword \
  -n backend
```

Expose MySQL:

```bash
oc expose deployment mysql \
  --port=3306 \
  --target-port=3306 \
  --name=mysql-service \
  -n backend
```

Verify:

```bash
oc get pods -n backend

oc get svc -n backend

oc get endpoints mysql-service -n backend
```

---

# 🖥️ Deploy Frontend Checker

Deploy the frontend connectivity test application:

```bash
oc create deployment frontend-checker \
  --image=kubernetesway/frontend-checker:v4 \
  -n frontend
```

Check the pod:

```bash
oc get pods -n frontend
```

View application logs:

```bash
oc logs deployment/frontend-checker -n frontend
```

When connectivity is available:

```text
✅ Successfully connected to
mysql-service.backend.svc.cluster.local:3306
```

---

# 🔒 NetworkPolicy Failure Simulation

The lab intentionally introduces a NetworkPolicy that prevents frontend access to MySQL.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-mysql
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: mysql
  policyTypes:
    - Ingress
```

Apply:

```bash
oc apply -f deny-mysql.yaml
```

Check:

```bash
oc get networkpolicy -n backend
```

The frontend application should eventually report:

```text
Unable to connect to mysql-service.backend.svc.cluster.local:3306
Error: timed out
```

---

# 📊 Prometheus Alerting

OpenShift User Workload Monitoring is used to monitor application workloads.

A `PrometheusRule` is configured in the application namespace.

Example CrashLoopBackOff alert:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: frontend-alerts
  namespace: frontend
spec:
  groups:

    - name: pod.rules
      rules:

        - alert: PodCrashLooping

          expr: |
            kube_pod_container_status_waiting_reason{
              namespace="frontend",
              reason="CrashLoopBackOff"
            } == 1

          for: 30s

          labels:
            severity: critical

          annotations:
            summary: "Pod is CrashLooping"
            description: "Container is restarting repeatedly"
```

Apply:

```bash
oc apply -f prometheus_rule.yaml
```

Verify:

```bash
oc get prometheusrule -n frontend
```

The metric can also be tested from:

```text
OpenShift Console
→ Observe
→ Metrics
```

using:

```promql
kube_pod_container_status_waiting_reason{
  namespace="frontend",
  reason="CrashLoopBackOff"
}
```

---

# 🔔 Alertmanager → AAP EDA

An `AlertmanagerConfig` routes application alerts to the AAP External Event Stream.

Verify the configuration:

```bash
oc get alertmanagerconfig -n frontend

oc get alertmanagerconfig -n demo
```

Example event received by EDA:

```json
{
  "status": "firing",
  "labels": {
    "alertname": "PodCrashLooping",
    "namespace": "frontend",
    "pod": "frontend-checker-xxxxx",
    "reason": "CrashLoopBackOff",
    "severity": "critical"
  }
}
```

---

# ⚡ Event-Driven Ansible Rulebook

The EDA rulebook matches the Prometheus alert and launches an AAP Workflow Template.

```yaml
---
- name: OpenShift Auto Recovery

  hosts: all

  sources:

    - ansible.eda.pg_listener:
        pg_port: 5432

  rules:

    - name: CrashLoopBackOff AI RCA

      condition: >-
        event.payload.alerts is defined and
        event.payload.alerts[0].status == "firing" and
        event.payload.alerts[0].labels.alertname == "PodCrashLooping" and
        event.payload.alerts[0].labels.namespace is defined and
        event.payload.alerts[0].labels.pod is defined

      action:

        run_workflow_template:

          name: gather_logs
          organization: Default

          job_args:

            extra_vars:

              namespace: "{{ event.payload.alerts[0].labels.namespace }}"

              pod_name: "{{ event.payload.alerts[0].labels.pod }}"

              alert_name: "{{ event.payload.alerts[0].labels.alertname }}"
```

This allows information contained in the Prometheus alert to dynamically populate the AAP workflow.

---

# 🔍 Collect OpenShift Diagnostics

The first stage of the AAP workflow collects troubleshooting evidence.

Example commands:

```bash
oc logs <pod> -n <namespace>

oc describe pod <pod> -n <namespace>

oc get events -n <namespace> \
  --sort-by=.metadata.creationTimestamp
```

For network-related incidents, additional information can be collected:

```bash
oc get networkpolicy -n frontend -o yaml

oc get networkpolicy -n backend -o yaml

oc get svc mysql-service -n backend -o yaml

oc get endpoints mysql-service -n backend -o yaml

oc get pods -n frontend --show-labels

oc get pods -n backend --show-labels
```

The evidence is passed to the next workflow stage using Ansible workflow artifacts.

---

# 🤖 AI Root Cause Analysis

The collected OpenShift evidence is sent to an AI RCA service.

The service is built using:

```text
Python
FastAPI
OpenAI API
```

The AI analyzes:

```text
Pod Logs
Pod Description
Kubernetes Events
Deployment Configuration
Service Configuration
Endpoints
NetworkPolicies
Labels / Selectors
```

The expected response is structured JSON.

Example:

```json
{
  "root_cause": "NetworkPolicy blocks frontend access to MySQL",
  "severity": "high",
  "confidence": 94,
  "recommended_action": "Allow TCP 3306 from frontend to backend MySQL",
  "network_policy_required": true,
  "network_policy": {
    "target_namespace": "backend",
    "target_app": "mysql",
    "source_namespace": "frontend",
    "source_app": "frontend-checker",
    "target_port": 3306,
    "protocol": "TCP"
  }
}
```

The AI recommendation is treated as input to automation — not as unrestricted authorization to modify the cluster.

---

# 🛠️ AI-Assisted NetworkPolicy Remediation

The AI-generated parameters are passed into an Ansible remediation job.

The resulting policy can resemble:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-frontend-to-mysql
  namespace: backend

spec:

  podSelector:
    matchLabels:
      app: mysql

  policyTypes:
    - Ingress

  ingress:

    - from:

        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend

          podSelector:
            matchLabels:
              app: frontend-checker

      ports:

        - protocol: TCP
          port: 3306
```

Apply:

```bash
oc apply -f allow-frontend-to-mysql.yaml
```

Verify:

```bash
oc get networkpolicy -n backend
```

Then check frontend logs:

```bash
oc logs deployment/frontend-checker -n frontend
```

Expected result:

```text
✅ Successfully connected to
mysql-service.backend.svc.cluster.local:3306
```

---

# 🔐 AAP OpenShift Service Account

A dedicated namespace and ServiceAccount can be created for AAP automation:

```bash
oc create namespace aap-automation
```

Create the account:

```bash
oc create serviceaccount aap-controller \
  -n aap-automation
```

For initial lab testing only:

```bash
oc adm policy add-cluster-role-to-user \
  cluster-admin \
  system:serviceaccount:aap-automation:aap-controller
```

Generate a temporary token:

```bash
oc create token aap-controller \
  -n aap-automation \
  --duration=3h
```

> ⚠️ `cluster-admin` is used only to simplify lab testing.
> A production implementation should use least-privilege RBAC.

---

# 🔐 Production RBAC Recommendation

For production, separate **diagnostic permissions** from **remediation permissions**.

The diagnostic service account should normally have read-only access to resources such as:

```text
pods
pods/log
events
services
endpoints
deployments
replicasets
networkpolicies
```

A separate remediation role can be authorized to modify only explicitly approved resources such as:

```text
networkpolicies
deployments/scale
```

This creates a safer separation between:

```text
AI Recommendation
        ↓
Validation
        ↓
Human / Policy Approval
        ↓
Authorized Remediation
```

---

# 🔄 AAP Workflow

The AAP Workflow Template orchestrates the incident lifecycle.

```text
EDA Alert
    ↓
Collect Diagnostics
    ↓
AI RCA
    ↓
Structured Recommendation
    ↓
Notification
    ↓
Optional Approval
    ↓
Remediation
    ↓
Post-Remediation Verification
```

Using a Workflow Template instead of a single Job Template makes it possible to add:

- Multiple diagnostic jobs
- AI analysis
- Manual approval
- Conditional remediation
- Teams / Slack / Email notifications
- Verification
- ServiceNow integration

---

# 🧪 Useful Troubleshooting Commands

Check pods:

```bash
oc get pods -A
```

Check a specific namespace:

```bash
oc get pods -n frontend
```

Check previous container logs:

```bash
oc logs <pod-name> -n frontend --previous
```

Describe a failing pod:

```bash
oc describe pod <pod-name> -n frontend
```

Check recent events:

```bash
oc get events -n frontend \
  --sort-by=.metadata.creationTimestamp
```

Check deployments:

```bash
oc get deployment -n frontend
```

Check services:

```bash
oc get svc -A
```

Check NetworkPolicies:

```bash
oc get networkpolicy -A
```

Check Prometheus rules:

```bash
oc get prometheusrule -A
```

Check Alertmanager configurations:

```bash
oc get alertmanagerconfig -A
```

---

# 🧠 Why AI RCA?

Traditional automation usually follows predefined logic:

```text
Alert X → Execute Script Y
```

AI-assisted RCA adds a reasoning layer:

```text
Alert
   ↓
Collect Evidence
   ↓
Analyze Multiple Signals
   ↓
Determine Probable Cause
   ↓
Generate Structured Recommendation
   ↓
Validate Recommendation
   ↓
Execute Approved Automation
```

This makes the solution useful for incidents where the same symptom — such as `CrashLoopBackOff` — can have many different root causes.

---

# 🚧 Future Enhancements

Planned enhancements include:

- Microsoft Teams notifications
- Slack notifications
- Email RCA reports
- ServiceNow incident creation
- Human-in-the-loop approval
- AI remediation confidence thresholds
- Post-remediation verification
- MCP server integration
- OpenShift API tools for AI agents
- Multiple LLM providers (OpenAI / Claude)
- Autonomous remediation with policy guardrails

---

# ⚠️ Security Considerations

This project is intended as an educational and portfolio lab.

For production environments:

- Do not grant AI systems unrestricted cluster access.
- Do not use permanent `cluster-admin` credentials.
- Use least-privilege RBAC.
- Store API keys and tokens in secrets or a secrets manager.
- Validate all AI-generated remediation parameters.
- Use server-side dry-run where possible.
- Require approval for high-risk changes.
- Maintain audit logs for automated actions.
- Treat Kubernetes logs and events as untrusted input.

---

# 🛠️ Technology Stack

| Component | Purpose |
|---|---|
| OpenShift | Kubernetes application platform |
| Prometheus | Monitoring and alert detection |
| Alertmanager | Alert routing |
| AAP EDA | Event-driven automation |
| AAP Workflow | Incident orchestration |
| Ansible | Diagnostics and remediation |
| FastAPI | AI RCA API service |
| OpenAI | Root cause analysis |
| NetworkPolicy | Kubernetes network security |

---

# 🎬 Project Series

### Part 1
Architecture and theory

### Part 2A
Prometheus + Alertmanager + AAP Event-Driven Ansible self-healing lab

### Part 2B
AI-powered Root Cause Analysis and intelligent remediation
https://www.youtube.com/watch?v=6u2qdhoYGig&t=782s
---

# ⭐ Project Goal

The goal of this project is to demonstrate how modern SRE and Platform Engineering teams can combine:

**Observability + Event-Driven Automation + AI + Human Governance**

to move from:

```text
Detect → Alert → Human Investigation
```

toward:

```text
Detect
  → Collect Evidence
  → AI RCA
  → Recommend
  → Approve
  → Remediate
  → Verify
```

This creates the foundation for an enterprise-grade **AI-assisted OpenShift incident response platform**.
