# Cloud Service Models & Platforms

---

## PaaS – I (Platform as a Service – Level I)

| Provider      | Services / Components                                              |
|---------------|--------------------------------------------------------------------|
| **AWS**       | Elastic Beanstalk, App Engine                                      |
| **Azure**     | App Service, Azure App Service                                     |
| **GCP**       | App Engine                                                         |
| **OpenShift** | OpenShift App Service                                              |

### Deployment Levels
| Level       | Description                                              |
|-------------|----------------------------------------------------------|
| PaaS – I    | Only source code upload                                  |
| PaaS – II   | Upload your container image                              |
| PaaS – III  | Upload source code + K8s manifest                        |

---

## PaaS – I (OpenStack)

| Component       | Description                          |
|-----------------|--------------------------------------|
| **Solum**       | App deployment service               |
| **Murano**      | App catalog / designer               |
| **Trove**       | Database as a Service                |
| **Designate**   | DNS as a Service                     |
| **Barbican**    | Key management service               |

---

## PaaS – II

| Platform / Component  | Description                                      |
|-----------------------|--------------------------------------------------|
| **Azure**             | Container Apps                                   |
| **AppRunner**         | AWS container service                            |
| **Cloud Run**         | GCP managed container service                    |
| **Magnum**            | OpenStack Kubernetes service                     |
| **HashiCorp**         | Infrastructure automation tools                  |
| **Rancher**           | Kubernetes management platform                   |
| **OpenStack**         | Magnum, Kuryr, Zun, Octavia                      |

### OpenStack PaaS-II Components
| Component    | Description                              |
|--------------|------------------------------------------|
| **Magnum**   | Container orchestration                  |
| **Kuryr**    | Container networking                     |
| **Zun**      | Container service                        |
| **Octavia**  | Load balancing service                   |

---

## PaaS – III

| Component     | Description                              |
|---------------|------------------------------------------|
| **Sahara**    | Big data / Hadoop service                |
| **Heat**      | Orchestration service                    |
| **Senlin**    | Clustering service                       |
| **Mistral**   | Workflow service                         |

### PaaS – III Cloud Providers
| Provider       | Service                                  |
|----------------|------------------------------------------|
| **OpenShift**  | OpenShift on OpenStack                   |
| **AWS**        | AWS Lambda (Serverless)                  |
| **Elastic**    | Elastic Lambda                           |

---

## IaaS (Infrastructure as a Service) – OpenStack Components

| Component       | Description                              |
|-----------------|------------------------------------------|
| **Nova**        | Compute service                          |
| **Cinder**      | Block storage service                    |
| **Neutron**     | Networking service                       |
| **Swift**       | Object storage service                   |
| **Glance**      | Image service                            |
| **Keystone**    | Identity / authentication service        |

---

## PaaS – Cloud Foundry & Others

| Platform             | Components / Description                                |
|----------------------|---------------------------------------------------------|
| **Cloud Foundry**    | Heroku, WS02 App Cloud, Engine Yard                     |
| **Heroku**           | PaaS with buildpacks, dynos                             |
| **WS02 App Cloud**   | Enterprise integration PaaS                             |
| **Engine Yard**      | Managed Rails/PHP hosting PaaS                          |

---

## PaaS – OpenStack (General)

| Component    | Description                              |
|--------------|------------------------------------------|
| **Solum**    | App deployment / CI pipeline             |
| **Magnum**   | Kubernetes / container orchestration     |
| **Murano**   | App catalog service                      |

---

## Apache Mesos

| Feature          | Description                                          |
|------------------|------------------------------------------------------|
| **Apache Mesos** | Distributed systems kernel / cluster manager         |

---

## SaaS (Software as a Service)

| Platform      | Description                              |
|---------------|------------------------------------------|
| **Trove**     | Database as a Service (OpenStack)        |
| **Sahara**    | Big Data as a Service (OpenStack)        |
| **Barbican**  | Key Management as a Service              |
| **Odoo**      | ERP / Business software SaaS             |
| **GitLab**    | DevOps / source code management SaaS     |

---

## Apache Hadoop / Spark

| Platform            | Description                                      |
|---------------------|--------------------------------------------------|
| **Apache Hadoop**   | Distributed big data framework                   |
| **Apache Spark**    | Fast in-memory big data processing               |

---

## Corrections & Notes

| Original (Handwritten) | Corrected          | Reason                                      |
|------------------------|--------------------|---------------------------------------------|
| Hero Ky                | **Heroku**         | Popular PaaS platform by Salesforce         |
| Barb Brian             | **Barbican**       | OpenStack key management service            |
| Octovia                | **Octavia**        | OpenStack load balancing service            |
| Kurys                  | **Kuryr**          | OpenStack container networking              |
| Mestral                | **Mistral**        | OpenStack workflow service                  |
| HashiCorp              | **HashiCorp**      | Correct – Vault, Terraform, Nomad           |
| Gitlabs                | **GitLab**         | Correct name is GitLab (no trailing 's')    |
| Elastes lambda         | **AWS Lambda**     | AWS serverless compute service              |
| w S02 App Cloud        | **WS02 App Cloud** | Enterprise integration PaaS                 |
| Barbican (in SaaS)     | Kept with note     | Barbican is OpenStack, not pure SaaS        |
