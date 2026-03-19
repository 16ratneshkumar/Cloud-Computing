# Cloud Service Models & Platforms

```
Developer Control  ←─────────────────────────────────────────────→  Provider Control
        │                                                                        │
      IaaS                    PaaS (I → II → III)                             SaaS
   (You manage OS,        (Provider manages runtime,              (Fully managed software,
    networking, VMs)       you deploy apps/containers)             you just use it)
```

---

## 1. IaaS — Infrastructure as a Service

> You manage: OS, runtime, middleware, apps. Provider manages: hardware, networking, virtualization.

### OpenStack IaaS Components

| Component      | Role                              |
|----------------|-----------------------------------|
| **Nova**       | Compute (VM provisioning)         |
| **Neutron**    | Networking                        |
| **Cinder**     | Block storage                     |
| **Swift**      | Object storage                    |
| **Glance**     | VM image registry                 |
| **Keystone**   | Identity & authentication         |

---

## 2. PaaS — Platform as a Service

> You manage: application code (and sometimes containers/manifests). Provider manages: everything below.

### PaaS Deployment Levels

```
PaaS – I    →   Upload only SOURCE CODE           (provider builds + runs it)
PaaS – II   →   Upload a CONTAINER IMAGE          (provider runs it)
PaaS – III  →   Upload SOURCE CODE + K8s MANIFEST (provider deploys to Kubernetes)
```

---

### 2.1 PaaS – Level I (Source Code Upload)

> Just push code — the platform handles build, runtime, scaling.

#### Public Cloud Providers

| Provider      | Service                    |
|---------------|----------------------------|
| **AWS**       | Elastic Beanstalk          |
| **Azure**     | App Service                |
| **GCP**       | App Engine                 |
| **OpenShift** | OpenShift App Service      |

#### Cloud Foundry Ecosystem (Open-Source PaaS-I)

| Platform           | Description                              |
|--------------------|------------------------------------------|
| **Cloud Foundry**  | Open-source PaaS foundation              |
| **Heroku**         | Cloud Foundry-based, buildpack model     |
| **WS02 App Cloud** | Enterprise integration PaaS              |
| **Engine Yard**    | Managed hosting PaaS (Rails/PHP)         |

#### OpenStack PaaS-I Components

| Component      | Role                                     |
|----------------|------------------------------------------|
| **Solum**      | App deployment & CI/CD pipeline          |
| **Murano**     | App catalog & designer                   |
| **Trove**      | Database as a Service                    |
| **Designate**  | DNS as a Service                         |
| **Barbican**   | Key & secrets management                 |

---

### 2.2 PaaS – Level II (Container Image Upload)

> Push a pre-built container — platform runs it on managed infrastructure.

#### Public Cloud Providers

| Provider        | Service                                  |
|-----------------|------------------------------------------|
| **AWS**         | App Runner                               |
| **Azure**       | Azure Container Apps                     |
| **GCP**         | Cloud Run                                |
| **HashiCorp**   | Nomad (container scheduling)             |
| **Rancher**     | Managed Kubernetes platform              |
| **Apache Mesos**| Distributed cluster manager             |

#### OpenStack PaaS-II Components

| Component     | Role                                     |
|---------------|------------------------------------------|
| **Magnum**    | Kubernetes / container orchestration     |
| **Zun**       | Bare container service (no K8s needed)   |
| **Kuryr**     | Container networking (bridges Neutron)   |
| **Octavia**   | Load balancing as a Service              |

---

### 2.3 PaaS – Level III (Source Code + K8s Manifest)

> Push code and Kubernetes manifests — platform builds and deploys to K8s.

#### Public Cloud Providers

| Provider       | Service                                  |
|----------------|------------------------------------------|
| **OpenShift**  | OpenShift on OpenStack                   |
| **AWS**        | AWS Lambda (Serverless / FaaS)           |
| **Elastic**    | Elastic / AWS Lambda integration         |

#### OpenStack PaaS-III Components

| Component     | Role                                     |
|---------------|------------------------------------------|
| **Heat**      | Orchestration (IaC for OpenStack)        |
| **Sahara**    | Big Data / Hadoop cluster service        |
| **Senlin**    | Clustering & auto-scaling service        |
| **Mistral**   | Workflow automation service              |

---

## 3. SaaS — Software as a Service

> Fully managed software delivered over the web. No infra or runtime management needed.

| Platform      | Category                          | Description                          |
|---------------|-----------------------------------|--------------------------------------|
| **Odoo**      | ERP / Business Suite              | Open-source all-in-one business app  |
| **GitLab**    | DevOps / SCM                      | Source control, CI/CD, issue tracking|
| **Trove**     | DBaaS (OpenStack-delivered SaaS)  | Managed database service             |
| **Sahara**    | Big Data SaaS (OpenStack)         | Managed Hadoop/Spark clusters        |
| **Barbican**  | Security SaaS (OpenStack)         | Secrets & key management             |

---

## 4. Flow Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CLOUD SERVICE MODELS                             │
├───────────────┬──────────────────────────────────────┬──────────────────┤
│     IaaS      │           PaaS (I / II / III)        │       SaaS       │
├───────────────┼──────────────────────────────────────┼──────────────────┤
│ Nova          │ I  → Elastic Beanstalk, App Engine   │ Odoo             │
│ Neutron       │     Heroku, Cloud Foundry, Solum     │ GitLab           │
│ Cinder        │                                      │ Trove (DBaaS)    │
│ Swift         │ II → App Runner, Cloud Run,          │ Sahara (BDaaS)   │
│ Glance        │     Magnum, Zun, Kuryr, Octavia      │ Barbican (KMaaS) │
│ Keystone      │                                      │                  │
│               │ III→ OpenShift, AWS Lambda,          │                  │
│               │     Heat, Sahara, Mistral, Senlin    │                  │
└───────────────┴──────────────────────────────────────┴──────────────────┘
```
