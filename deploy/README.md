# 

# **Carbide Installation**

| Revision | Date | Notes |
| :---- | :---- | :---- |
| 0.1 |  |  |
|  |  |  |

## Context

Below is a **prescriptive, BYO‑Kubernetes bring‑up guide** that is **not site‑specific**. It encodes the **order of operations** and the **exact versions/images we verified**, plus what you must configure if you already operate some of the common services yourself.

**Principles**

* All unknown values remain explicit placeholders like \<REPLACE\_ME\>.

* If you **already run** one of the core services (PostgreSQL, Vault, cert‑manager, Temporal), do the “**If you already have it**” checklist for that service.

* If you **don’t**, deploy the **“Reference version we ran”** (images and versions below) and apply the config under “**If you deploy our reference**”.

---

## **0\) Validated baseline (what we ran)**

**Kubernetes & node runtime**

* Control plane: **Kubernetes v1.30.4** (server)  
* Nodes: kubelet **v1.26.15**, container runtime **containerd 1.7.1**  
* CNI: **Calico v3.28.1** (node & controllers)  
* OS: Ubuntu **24.04.1 LTS**

**Networking**

* Ingress: **Project Contour v1.25.2** (controller) \+ **Envoy v1.26.4** (daemonset)  
* Load balancer: **MetalLB v0.14.5** (controller & speaker)

**Secret & cert plumbing**

* **External Secrets Operator v0.8.6**  
* **cert‑manager v1.11.1** (controller/webhook/CA‑injector)  
  * **Approver‑policy v0.6.3**  
     (Pods present as cert-manager, cainjector, webhook, and policy controller.  )

**State & identity**

* **PostgreSQL**: Zalando **Postgres Operator v1.10.1** \+ **Spilo‑15 image 3.0‑p1** (Postgres **15**)  
* **Vault**: **Vault server 1.14.0**, **vault‑k8s injector 1.2.1**

**Temporal & search**

* **Temporal server 1.22.6** (frontend/history/matching/worker)  
  * **Admin tools 1.22.4**, **UI 2.16.2**  
* **Elasticsearch 7.17.3** (for Temporal visibility)

**Monitoring & telemetry (reference stack \-  OPTIONAL)**

* **Prometheus Operator v0.68.0**; **Prometheus v2.47.0**; **Alertmanager v0.26.0**  
* **Grafana v10.1.2**; **kube‑state‑metrics v2.10.0**  
* **OpenTelemetry Collector v0.102.1**  
* **Loki v2.8.4**  
* **Node exporter v1.6.1**

**Carbide components (what this guide installs)**

* **Carbide core (forge‑system)**  
  * nvmetal-carbide:v2025.07.04-rc2-0-8-g077781771 (primary carbide-api, plus supporting workloads)  
* **cloud‑api**: nvcr.io/nvidian/nvforge-devel/cloud-api:v0.2.72 (two replicas)  
* **cloud‑workflow**: nvcr.io/nvidian/nvforge-devel/cloud-workflow:v0.2.30 (cloud‑worker, site‑worker)  
* **cloud‑cert‑manager (credsmgr)**: nvcr.io/nvidian/nvforge-devel/cloud-cert-manager:v0.1.16  
* **elektra-site-agent**: nvcr.io/nvidian/nvforge-devel/forge-elektra:v2025.06.20-rc1-0

---

## **1\) Order of operations (high level)**

1. **Cluster & networking ready**  
   * Kubernetes v1.26–1.30 (tested on 1.30.4), containerd 1.7.x, Calico (or conformant CNI)  
   * Ingress controller (Contour/Envoy) \+ LoadBalancer (MetalLB or cloud LB) available  
   * Available DNS recursive resolvers and NTP available

2. **Foundation services (in this order)**  
   * **External Secrets Operator (ESO) \- Optional**  
   * **cert‑manager** (Issuers/ClusterIssuers in place)  
   * **PostgreSQL** (DB/role/extension prerequisites below)  
   * **Vault** (PKI engine, K8s auth, policies/paths)  
   * **Temporal** (server up; register namespaces)

3. **Carbide core (forge‑system)**  
   * Bring up carbide-api & supporting services (DHCP/PXE/DNS/NTP as required)  
4. **Carbide Rest components**  
   * Deploy **cloud‑api**, **cloud‑workflow (cloud‑worker & site‑worker)**, **cloud‑cert‑manager (credsmgr)**  
   * **Seed DB and register Temporal namespaces** (cloud, site, then the **site UUID**)  
   * **Create OTP & bootstrap secrets** for **elektra‑site‑agent**; roll restart it

5. **Monitoring (optional but recommended)**  
   * Prometheus operator, Grafana, Loki, OTel, node exporter

The rest of this document details **what to configure** for each shared service and **how the NVCarbide workloads use them**.  
---

## **2\) External Secrets Operator (ESO)**

**Reference version we ran:** ghcr.io/external-secrets/external-secrets:v0.8.6.

**What you must provide**

* A SecretStore/ClusterSecretStore pointing at **Vault** and, if applicable, a Postgres secret namespace.  
* ExternalSecret objects similar to these (namespaces vary by component):  
  * forge-roots-eso → target secret forge-roots with keys site-root, forge-root  
  * DB credentials ExternalSecrets per namespace (e.g., clouddb-db-eso → forge.forge-pg-cluster.credentials)  
* Ensure an **image pull secret** (e.g., imagepullsecret) exists in namespaces that pull from nvcr.io.

---

## **3\) cert‑manager (TLS & trust)**

**Reference versions we ran**

* Controller/Webhook/CAInjector **v1.11.1**  
* Approver‑policy **v0.6.3**  
* ClusterIssuers present: self-issuer, site-issuer, vault-issuer, vault-forge-issuer

**If you already have cert‑manager**

* Ensure it is ≥ **v1.11.1** and that:  
  * Your **ClusterIssuer** objects can issue:  
    * cluster internal certs (service DNS SANs) and  
    * any externally‑facing FQDNs you choose.  
  * Approver flows allow your teams to create Certificate resources for the NVCarbide namespaces.

**If you deploy our reference**

* Install cert‑manager **v1.11.1** and **approver‑policy v0.6.3**.  
* Create ClusterIssuers matching your PKI: \<ISSUER\_NAME\>.  
* Typical **SANs** for NVFORGE services include:  
  * Internal service names (e.g., carbide-api.\<ns\>.svc.cluster.local, carbide-api.forge)  
  * Optional external FQDNs (your domains)

---

## **4\) Vault (PKI & secrets)**

**Reference versions we ran**

* Vault server **1.14.0** (HA Raft)  
* Vault injector (**vault‑k8s**) **1.2.1**

**If you already have Vault**

* **Enable PKI** engine(s) for:  
  * Root/intermediate CA chain used by NVFORGE components (where your forge-roots/site-root derive).  
* **Enable K8s auth** at path auth/kubernetes and create roles that map service accounts in:  
  * forge-system, cert-manager, cloud-api, cloud-workflow, elektra-site-agent

* **Policies/paths** (indicative):  
  * KV v2 for application material: \<VAULT\_PATH\_PREFIX\>/kv/\*  
  * PKI for issuance: \<VAULT\_PATH\_PREFIX\>/pki/\*

**If you deploy our reference**

* Stand up Vault **1.14.0** with TLS (server cert for vault.vault.svc).  
* Configure:  
  * VAULT\_ADDR (cluster‑internal URL, e.g., https://vault.vault.svc:8200 or http://vault.vault.svc:8200 if testing)  
  * KV mounts and PKI roles. Components expect env like:  
    * VAULT\_PKI\_MOUNT\_LOCATION, VAULT\_KV\_MOUNT\_LOCATION, VAULT\_PKI\_ROLE\_NAME=forge-cluster

* Injector (optional) may be enabled for sidecar‑based secret injection.


**Used by components**

* **carbide‑api** consumes Vault for PKI and secrets (env VAULT\_\*).  
* **credsmgr** interacts with Vault for CA material exposed to the site bootstrap flow.

---

## **5\) PostgreSQL (DB)**

**Reference we ran**

* **Zalando Postgres Operator v1.10.1**  
* **Spilo‑15 image 3.0‑p1** (Postgres **15**)

**If you already have Postgres**

* Provide a database \<POSTGRES\_DB\> and role \<POSTGRES\_USER\> with password \<POSTGRES\_PASSWORD\>.  
* Enable **TLS** (recommended) or allow secure network policy between DB and the NVCarbide namespaces.  
* Create extensions (the apps expect these):

| CREATE EXTENSION IF NOT EXISTS btree\_gin;CREATE EXTENSION IF NOT EXISTS pg\_trgm; |
| :---- |

This can be done with a call similar to the following:

| psql "postgres://\<POSTGRES\_USER\>:\<POSTGRES\_PASSWORD\>@\<POSTGRES\_HOST\>:\<POSTGRES\_PORT\>/\<POSTGRES\_DB\>?sslmode=\<POSTGRES\_SSLMODE\>" \\     \-c 'CREATE EXTENSION IF NOT EXISTS btree\_gin;' \\     \-c 'CREATE EXTENSION IF NOT EXISTS pg\_trgm;' |
| :---- |

* Make the DSN available to workloads via ESO targets (per‑namespace credentials):  
  * Examples: forge.forge-pg-cluster.credentials, forge-system.carbide.forge-pg-cluster.credentials, elektra-site-agent.elektra.forge-pg-cluster.credentials (names are examples—use your own).

**If you deploy our reference**

* Deploy the Zalando operator and a Spilo‑15 cluster sized for your SLOs.  
  Expose a ClusterIP service on **5432** and surface credentials through ExternalSecrets to each namespace that needs them.

---

## **6\) Temporal**

**Reference we ran**

* Temporal server **1.22.6** (frontend/history/matching/worker)  
* UI **2.16.2**, admin tools **1.22.4**  
* Frontend service endpoint (cluster‑internal):  
   **temporal-frontend.temporal.svc:7233**

**Required namespaces**

* Base: **cloud, site**  
* Per‑site: the **site UUID** 

**If you already have Temporal**

* Ensure the **frontend gRPC endpoint** is reachable from NVCarbide workloads and present the proper **mTLS**/CA if you require TLS.  
* Register namespaces:

| tctl \--ns cloud namespace registertctl \--ns site  namespace registertctl \--ns \<SITE\_UUID\> namespace register (once you know the site UUID) |
| :---- |

**If you deploy our reference**

* Deploy Temporal as above, expose :7233.  
* Register the same namespaces.

---

## **7\) What each carbide workload expects \- exact images we used and what resources needs to be applied in what order:**

## **7.0 Site CA secrets for cloud‑cert‑manager (namespace cert-manager) \- Reference Only**

#### Goal

Before deploying **cloud‑cert‑manager / credsmgr**, we need a *site‑specific root CA* that Vault will use to issue certificates.

Concretely, this means **two Secrets in the cert-manager namespace**:

* **vault-root-ca-certificate**  
  * data.certificate → PEM‑encoded root CA certificate  
* **vault-root-ca-private-key**  
  * data.privatekey → PEM‑encoded root CA private key

Customers can create these however they want (their own PKI, HSM, etc.).

For internal use, SAs can shortcut this with the gen-site-ca.sh helper script in the **stardrive** repo.

---

### **7.0.1 SA helper: gen-site-ca.sh (stardrive)**

From the root of the [stardrive](https://gitlab-master.nvidia.com/nvmetal/stardrive) repo:

| ./scripts/cloud/gen-site-ca.sh \<SITE\_DIR\> |
| :---- |

For quick testing, you can simply run:

| ./scripts/cloud/gen-site-ca.sh . |
| :---- |

**High‑level behavior (no need to explain in detail to customers):**

* Generates a **self‑signed RSA 4096 root CA**, valid for 10 years.  
* Uses kubectl create secret ... \--dry-run=client to build Secret YAMLs for:  
  * vault-root-ca-certificate (with certificate: \<PEM\>),  
  * vault-root-ca-private-key (with privatekey: \<PEM\>),  
     **both in namespace cert-manager.**  
* Writes those YAMLs under:  
  * \<SITE\_DIR\>/cloud/secrets/vault-root-ca-certificate.enc.yaml  
  * \<SITE\_DIR\>/cloud/secrets/vault-root-ca-private-key.enc.yaml  
* Also writes ksops glue (cloud/secret-generator.yaml, cloud/kustomization.yaml) which are for internal templating and **can be ignored** for BYO/manual setups.

For our SA workflow we typically **do not** encrypt these for BYO demos; we just apply them directly

### **7.0.2 Creating the Secrets in the cluster**

Once gen-site-ca.sh has run, create the Secrets in the cluster:

| cd \<SITE\_DIR\>/cloud/secretsmv ./vault-root-ca-certificate.enc.yaml ./vault-root-ca-certificate.yaml  kubectl apply \-f vault-root-ca-certificate.yamlmv ./vault-root-ca-private-key.enc.yaml .vault-root-ca-private-key.yaml  kubectl apply \-f vault-root-ca-private-key.yaml |
| :---- |

This results in:

* vault-root-ca-certificate (in namespace cert-manager)  
* vault-root-ca-private-key (in namespace cert-manager)

with the certificate and key generated by the script.

If a customer already has their own root CA and doesn’t want us to generate one, they can skip the script entirely and instead run, for example:

| kubectl \-n cert-manager create secret generic vault-root-ca-certificate \\  \--from-file=certificate=./cacert.pemkubectl \-n cert-manager create secret generic vault-root-ca-private-key \\  \--from-file=privatekey=./ca.key |
| :---- |

As long as the **secret names** (vault-root-ca-certificate, vault-root-ca-private-key) and **key names** (certificate, privatekey) match, the rest of the guide (cloud‑cert‑manager / credsmgr, ClusterIssuer vault-issuer, etc.) will work as written.

## 

## **7.1 cloud‑cert‑manager “credsmgr” (namespace cert-manager)**

Branch with These changes \-\> [link](https://gitlab-master.nvidia.com/nvmetal/cloud-cert-manager/-/merge_requests/59)

### **Role**

cloud-cert-manager (the **credsmgr** deployment \+ related RBAC/issuers) is  responsible for issuing mTLS certificates for temporal access 

* Runs an embedded vault process that acts as a self contained certificate issuer  
* Exposes a service (credsmgr) that can issue certificates  
* Applies a **CertificateRequestPolicy** and RBAC so only certain namespaces (temporal, cloud-db, cloud-workflow, cloud-api) can obtain certs through the service.

### **7.1.1 Kustomize layout (sa-enablement)**

Under the cloud-cert-manager repo:

| deploy/kustomize/   cert-approval-policy.yaml    \# CertificateRequestPolicy (credsmgr-policy-approver)   cluster-issuer.yaml          \# ClusterIssuer "vault-issuer"   cluster-role.yaml            \# ClusterRole "credsmgr-cluster-role"   cluster-role-binding.yaml    \# ClusterRoleBinding "credsmgr-cluster-role-binding"   deployment.yaml              \# credsmgr \+ embedded Vault containers   service.yaml                 \# Service on ports 8000 (credsmgr) & 8200 (vault)   namespace.yaml               \# namespace definition (cert-manager)   sa.yaml                      \# credsmgr ServiceAccount   sa-role.yaml                 \# Role for secrets in namespace   sa-rolebinding.yaml          \# binds SA to Role   vault-config-configmap.yaml  \# Vault server configuration   kustomization.yaml           \# Kustomize entrypoint  |
| :---- |

### **7.1.2 Behavior in our reference environment**

* **Namespace:** cert-manager  
* **ClusterIssuer:** vault-issuer  
  * spec.vault.server \= http://credsmgr:8200  
  * spec.vault.path   \= pki/sign/cloud-cert  
  * Token for this Issuer is read from K8s Secret vault-token (key token) in cert-manager.  
* **CertificateRequestPolicy:** **credsmgr-policy-approver**  
  * Allows requests using issuerRef (ClusterIssuer vault-issuer) originating from:  
    * temporal, cloud-db, cloud-workflow, cloud-api namespaces.  
  * RBAC binds cert-manager ServiceAccount to a ClusterRole that can use that policy and manage relevant Certificate/CertificateRequest resources.  
* **credsmgr Deployment:**  
  * Container credsmgr:  
    * Image placeholder: credsmgr (overridden in kustomization.yaml to an internal image; customers might need to  build their own).  
    * Env:  
      * FORGE\_VAULT\_INGRESS\_URL → https://temporal.forge.nvidia.com (reference; customer-specific).  
      * FORGE\_CERT\_MANAGER\_SECRET\_NAME → "vault-token" (Vault token secret name).  
      * CERT\_MANAGER\_NS → "cert-manager".  
    * Probes on port **8001** (/healthz), but the service port for credsmgr itself is **8000**.  
    * Mounts:  
      * /vault/secrets/token → vault-token (emptyDir; credsmgr populates token).  
      * /vault/secrets/vault-root-ca-certificate → Secret vault-root-ca-certificate.  
      * /vault/secrets/vault-root-ca-private-key → Secret vault-root-ca-private-key.  
  * Container vault:  
    * Image placeholder: vault (overridden in kustomization.yaml to nvcr.io/.../vault:1.19.0 in the reference).  
    * Uses vault-server-config ConfigMap as Vault config (file storage, UI, telemetry).  
    * Serves HTTP on 8200/8201/8202 with TLS disabled in the sample config.

* **Service:** credsmgr (ClusterIP) in cert-manager  
  * Port 8000 named credsmgr → used by clients for CA functions.  
  * Port 8200 named vault   → used by ClusterIssuer vault-issuer as spec.vault.server.

### **7.3.3 Customer knobs (what SAs must adjust, and where)**

#### **1\. Images & registry**

**File:** deploy/kustomize/kustomization.yaml

Current internal overrides:

| images:  \- name: credsmgr    newName: nvcr.io/nvidian/nvforge-devel/cloud-cert-manager    newTag: v0.1.16  \- name: vault    newName: nvcr.io/nvidian/nvforge-devel/vault    newTag: 1.19.0 |
| :---- |

For customers:

1. They must **build credsmgr** from source and push to their registry:

| docker build \-t \<REGISTRY\>/\<PROJECT\>/cloud-cert-manager:\<TAG\> .docker push \<REGISTRY\>/\<PROJECT\>/cloud-cert-manager:\<TAG\> |
| :---- |

2. If they use an embedded Vault container (as in the reference), they must also build/push a Vault image or reference an approved Vault image from their environment.  
3. Update kustomization.yaml:

| images:  \- name: credsmgr    newName: \<REGISTRY\>/\<PROJECT\>/cloud-cert-manager    newTag: \<TAG\>  \- name: vault    newName: \<REGISTRY\>/\<PROJECT\>/vault  \# or their chosen path    newTag: \<VAULT\_TAG\> |
| :---- |

4. Ensure their registry auth is wired:  
   * Add or adjust imagePullSecrets in credsmgr-deployment.yaml if needed.

#### **2\. Vault path & server (ClusterIssuer)**

**File:** deploy/kustomize/cluster-issuer.yaml

Reference config:

*  spec.vault.server: http://credsmgr:8200  
* spec.vault.path: pki/sign/cloud-cert  
* spec.vault.auth.tokenSecretRef.name: vault-token

#### Customer concerns:

* server must point at the Vault endpoint that vault-issuer should use:  
  * If they keep the embedded Vault sidecar, http://credsmgr:8200 is fine.  
  * If they point to an **external Vault cluster**, this must be updated to that URL.  
* path must match the actual PKI role path in their Vault setup (e.g., pki\_int/sign/\<role\>).  
* tokenSecretRef.name and key must match the **secret name and key** where the Vault token is stored.

**3\. Vault configuration**

**File:** deploy/kustomize/vault-config-configmap.yaml

Reference config:

* listener "tcp" block with:  
  * tls\_disable \= 1  
  * address \= "\[::\]:8200"  
  * cluster\_address \= "\[::\]:8201"  
* storage "file" using /vault/data.

Customer changes:

* If they want **TLS on Vault**:  
  * Adjust listener to enable TLS and mount certs appropriately.  
* If they want a different **storage backend** (e.g. Raft, cloud KMS auto-unseal):  
  * Replace the storage "file" block with their chosen storage stanza.  
* They may also tune telemetry or remove ui \= true according to security requirements.

#### **4\. CertificateRequestPolicy & RBAC**

**Files:**

* deploy/kustomize/cert-approval-policy.yaml  
* deploy/kustomize/cluster-role.yaml  
* deploy/kustomize/cluster-role-binding.yaml

Reference behavior:

* CertificateRequestPolicy named credsmgr-policy-approver:  
  * Applies to ClusterIssuer vault-issuer.  
  * Restricts usage to namespaces: temporal, cloud-db, cloud-workflow, cloud-api.  
* ClusterRole/Binding give the cert-manager ServiceAccount the policy.cert-manager.io/certificaterequestpolicies use verb and allow it to manage Certificate/CertificateRequest resources.

**Customer adjustments:**

* Confirm that subject.organizations, ipAddresses, and usages are aligned with their PKI policies.  
* Retain the ClusterRole/Binding mapping to the cert-manager ServiceAccount unless they have a different trust model.

#### **5\. Vault token and root CA secrets**

**Files:**

* cluster-issuer.yaml (references vault-token)  
* deployment.yaml (references vault-root-ca-certificate, vault-root-ca-private-key)

Customer must ensure:

* A Secret named vault-token exists in the chosen namespace with:  
  * data.token containing a token with the capabilities required for the PKI path.  
* Secrets vault-root-ca-certificate and vault-root-ca-private-key exist (or the references are removed if not needed in their design).

### **7.1.4 How to apply cloud-cert-manager / credsmgr**

Once:

* Images for credsmgr and vault are built and pushed,  
* Vault PKI roles/paths and token secrets are configured,  
* The CertificateRequestPolicy namespaces are adjusted,

the architect can deploy this component with:

| kubectl apply \-k ./deploy/kustomize |
| :---- |

This will create:

* The credsmgr ServiceAccount / Role / RoleBinding,  
* The Vault configuration ConfigMap,  
* The credsmgr \+ Vault Deployment and Service,  
* The Vault-backed ClusterIssuer (vault-issuer),  
* The CertificateRequestPolicy and its RBAC bindings.

After that, cert-manager should be able to issue certificates for the designated namespaces by talking to Vault through credsmgr, using the policies and paths defined here.

## **7.2 Install Temporal Certificates \- Reference only\!**

Branch with These changes \-\> [link](https://gitlab-master.nvidia.com/nvmetal/carbide-external/-/tree/sa-enablement/apps/cloud-certificates?ref_type=heads)

This step pre‑provisions the TLS certs that Temporal and the cloud workloads use for mTLS:

* **Client certs** used by cloud-api and cloud-workflow when calling Temporal.  
* **Server certs** used by Temporal for its cloud, site, and inter‑service endpoints.

All of these are issued by the vault-issuer ClusterIssuer you created in **7.1 cloud‑cert‑manager** and ultimately chain back to the site root CA secrets from **7.0**.

This section is **internal only**: CNs/SANs are hard‑coded to \*.temporal.nvidia.com. Customers should treat this as a pattern and replace hostnames and namespaces with their own.  
---

### **7.2.1 Kustomize layout**

From the internal repo location you already referenced for SAs, there is a Kustomize base with:

| client-cloud-api.yaml         \# client cert for cloud-apiclient-cloud-workflow.yaml    \# client cert for cloud-workflowserver-cloud.yaml             \# server cert for "cloud" Temporal endpointserver-interservice.yaml      \# server cert for Temporal inter-service trafficserver-site.yaml              \# server cert for "site" Temporal endpointkustomization.yaml            \# references all of the above |
| :---- |

kustomization.yaml simply lists these five Certificate resources as resources

### **7.2.2 What the certs do**

**Client certificates**

* **client-cloud-api.yaml**  
  * Namespace: cloud-api  
  * Resource: Certificate temporal-client-cloud-certs  
  * Produces Secret: temporal-client-cloud-certs  
  * CN / DNS:  
    * commonName: cloud.client.temporal.nvidia.com  
    * dnsNames: \[cloud.client.temporal.nvidia.com\]  
  * Intended use: mTLS client cert for the **cloud‑api** deployment to call Temporal.  
* **client-cloud-workflow.yaml**  
  * Namespace: cloud-workflow  
  * Resource: Certificate temporal-client-cloud-certs  
  * Produces Secret: temporal-client-cloud-certs  
  * CN / DNS identical to the cloud‑api client cert.  
  * Intended use: mTLS client cert for **cloud-worker** and **site-worker** to call Temporal.

Both client certs:

* issuerRef: name: vault-issuer, kind: ClusterIssuer, group: cert-manager.io  
* usages: server auth, client auth  
* Duration: 2160h (90 days), renewBefore: 360h (15 days)

Cloud‑side deployments mount these Secrets at:

* /var/secrets/temporal/certs/client-cloud/ in cloud-api  
* /var/secrets/temporal/certs/client-cloud/ in cloud-workflow

---

**Server certificates (Temporal namespace)**

All three server certs live in **namespace temporal** and are issued by vault-issuer with usages: \[server auth, client auth\].

* **server-cloud.yaml**  
  * Certificate server-cloud-certs  
  * Secret: server-cloud-certs  
  * CN / DNS:  
    * cloud.server.temporal.nvidia.com  
  * Duration: 2h, renewBefore: 1h  
     (short lifetime for internal/dev testing; adjust if you need something longer‑lived.)  
* **server-interservice.yaml**  
  * Certificate server-interservice-certs  
  * Secret: server-interservice-certs  
  * CN / DNS:  
    * interservice.server.temporal.nvidia.com  
  * Duration: 2160h / renewBefore: 360h  
* **server-site.yaml**  
  * Certificate server-site-certs  
  * Secret: server-site-certs  
  * CN / DNS:  
    * site.server.temporal.nvidia.com  
  * Duration: 2160h / renewBefore: 360h

These Secrets are consumed by your Temporal deployment (frontends, gateways, etc.) according to your internal Temporal Helm values / manifests.

### **7.2.3 Applying the cloud certs**

**Prerequisites:**

* 7.0 root CA secrets (vault-root-ca-certificate, vault-root-ca-private-key) exist in cert-manager.  
* 7.1 cloud-cert-manager / credsmgr and the vault-issuer ClusterIssuer are healthy.

From the directory containing the kustomization.yaml and the five cert YAMLs above, apply:

| kubectl apply \-k \<path-to-cloud-certs-kustomize\> |
| :---- |

This will create:

* temporal-client-cloud-certs in namespaces:  
  * cloud-api  
  * cloud-workflow  
* server-cloud-certs, server-interservice-certs, server-site-certs in namespace temporal.

You can verify issuance with:

| kubectl \-n cloud-api        get certificate,secret temporal-client-cloud-certskubectl \-n cloud-workflow   get certificate,secret temporal-client-cloud-certskubectl \-n temporal         get certificate,secret server-\*-certs |
| :---- |

## **7.3 Install Temporal (reference only)**

Branch with These changes \-\> [link](https://gitlab-master.nvidia.com/nvmetal/temporal-helm-charts/-/merge_requests/35)  
**Audience note:**  
This section is **for internal SAs only**. Customers are expected to deploy Temporal themselves (self‑hosted or managed/hosted), following Temporal’s public documentation. They do **not** need to copy our values file; they just need to satisfy the integration constraints and the certs from **7.2**.

### **7.3.1 What we ran internally**

We used the internal temporal-helm-charts/temporal chart with a custom values.yaml that:

* Deploys Temporal **server 1.22.6** (temporalio/server:1.22.6).  
* Enables:  
  * admintools (temporalio/admin-tools:1.22.4)  
  * web (temporalio/ui:2.16.2)  
  * Embedded **Elasticsearch 7.17.3** in the same namespace for visibility.  
* Uses **PostgreSQL** for both persistence and visibility:  
  * Main DB: temporal as user temporal\_user  
    * Host: forge-pg-cluster.postgres.svc.cluster.local:5432  
    * Credentials from Secret temporal-user.forge-pg-cluster.credentials (key password)  
  * Visibility DB: temporal\_visibility as user temporal\_visibility\_user  
    * Credentials from Secret temporal-visibility-user.forge-pg-cluster.credentials  
* Enables **TLS everywhere** using the certs created in **7.2**:  
  * Mounts these secrets:  
    * server-interservice-certs  
    * server-cloud-certs  
    * server-site-certs  
  * Configures global.tls:  
    * Internode server/client use server-interservice-certs  
    * Frontend hostOverrides:  
      * interservice.server.temporal.nvidia.com → server-interservice-certs  
      * cloud.server.temporal.nvidia.com       → server-cloud-certs  
      * site.server.temporal.nvidia.com        → server-site-certs  
  * CLI and UI (admintools \+ web) set:  
    * \*\_TLS\_SERVER\_NAME=interservice.server.temporal.nvidia.com  
    * \*\_TLS\_CERT/TLS\_KEY/TLS\_CA pointing at /var/secrets/temporal/certs/server-interservice/\*  
* Exposes the **frontend** on a ClusterIP service on port **7233** in namespace temporal (internal only).

We installed it with a simple Helm command from the chart directory, e.g.:

| helm dependency update ./ helm upgrade \--install temporal . \\  \--namespace temporal \\  \-f ./values.yaml |
| :---- |

After a successful install in our reference cluster, kubectl \-n temporal get pods showed:

| elasticsearch-master-0temporal-admintools-...temporal-frontend-...temporal-history-...temporal-matching-...temporal-web-...temporal-worker-... |
| :---- |

All in Running state.

* In the customer environment, they can choose any hostnames / ports, as long as:

  * cloud-api and cloud-workflow configs point at the correct host:port.

  * The TLS settings and CA chain match the secrets they configure (analogous to the certs in **7.2**).

### **7.3.3 Ensure base Temporal namespaces (cloud and site)**

Once the Temporal cluster is up and all pods are Running, two **Temporal namespaces** must exist for Carbide:

* cloud  
* site

Customers can create these namespaces however they like (e.g. tctl namespace register, the Temporal Web UI, or automation).

For **our SAs**, the temporal-helm-charts repo includes a helper script that does this idempotently:

* **Location (repo root):** ./register-temporal-namespaces.sh  
  **Optional env:** set KUBECTL\_CONTEXT if you need a non‑default kube context.

What the script does at a high level:

1. Waits for the **admintools** pod in namespace temporal to be Ready.  
2. Waits for at least one **worker** pod to be Ready.  
3. For each namespace in cloud and site:  
   * Runs tctl \--ns \<NS\> namespace describe inside the admintools pod.  
   * If it doesn’t exist, runs tctl \--ns \<NS\> namespace register.  
   * Retries up to 10 times with a 120‑second backoff.

Example invocation and (trimmed) output:

| ./register-temporal-namespaces.sh⏳  Waiting for Temporal admintools pod to be Ready...✅  admintools pod temporal-admintools-... is Ready⏳  Waiting for at least one Temporal worker pod to be Ready...✅  worker pod temporal-worker-... is ReadyCreating Temporal namespace 'cloud' (attempt 1/10)...✓  Namespace 'cloud' created.Creating Temporal namespace 'site' (attempt 1/10)...✓  Namespace 'site' created.🎉 Temporal namespaces 'cloud' and 'site' ensured. |
| :---- |

For customers, the equivalent manual commands (run wherever tctl can reach their Temporal frontend) are:

| tctl \--ns cloud namespace registertctl \--ns site  namespace register |
| :---- |

## 

## **7.4 cloud-db (namespace cloud-db)**

Branch with These changes \-\> [link](https://gitlab-master.nvidia.com/nvmetal/cloud-db/-/merge_requests/524)

#### Role

cloud-db is a **migration job** that initializes or upgrades the cloud database schema. It’s **not** a long-running service: it runs once with a given version, applies migrations to Postgres, and exits.

#### Reference behavior we run internally

* **Namespace:** cloud-db  
* **Workload type:** one-off Job (e.g., cloud-db-migration-0-1-45)  
* **Image / version (reference only):**  
   nvcr.io/nvidian/nvforge-devel/cloud-db:v0.1.45  
   In customer environments, this must be replaced with an image they build from the GitHub repo and push to their own registry.  
* **Config source:** a ConfigMap with:  
  * dbHost → hostname/FQDN of the Postgres service  
  * dbPort → Postgres port (typically 5432\)  
  * dbName → name of the application database (e.g., forge or cloud)  
  * dbUser → logical Postgres user (e.g., forge)  
* **Credentials source:** a single DB credential Secret (in our reference: forge.forge-pg-cluster.credentials) with keys:  
  * username  
  * password  
* **Runtime behavior:**  
  * On start, the Job logs:  
    * PGUSER, DBNAME and a masked DB URL: postgres://PGUSER:\*\*\*\*@PGHOST:PGPORT/DBNAME  
  * If PGUSER, PGPASSWORD, or DBNAME are missing, the Job fails immediately.  
  * Otherwise, it executes a migration entrypoint (e.g., /usr/local/bin/migrate.sh).

### **What a solution architect must adjust for a customer environment**

When you hand this to a customer, they need to **review and customize** the following fields:

1. **Postgres host & port**  
   * **dbHost** must be set to **their Postgres service FQDN or hostname**, most likely not our internal forge-pg-cluster.postgres.svc.cluster.local.  
   * **dbPort** must match their DB port (usually still 5432).  
2. **Database name & user**  
   * **dbName** should be the name of the database where cloud schemas live (e.g., cloud, forge, or whatever they use).  
   * **dbUser** should match the Postgres user they’ve provisioned for cloud components.  
3. **Database credential Secret**  
   * The Job expects to read PGUSER and PGPASSWORD from a K8s Secret. In our reference, that Secret is forge.forge-pg-cluster.credentials with keys username and password.  
   * Customers can either:  
     * Reuse that Secret name (and ESO mapping) and just make sure the values are correct for their DB, **or**  
     * Change the Job’s environment to point at **their own Secret name** (e.g., cloud-db-credentials) while preserving the key names (username / password).  
4. **Image and registry**  
   * The **image name and tag** must be updated to point at the **customer’s registry**, not nvcr.io.  
   * For example:  
     * Build from the repo: docker build \-t \<REGISTRY\>/\<PROJECT\>/cloud-db:\<TAG\> .  
     * Push it: docker push \<REGISTRY\>/\<PROJECT\>/cloud-db:\<TAG\>  
     * Update the deployment to use image: \<REGISTRY\>/\<PROJECT\>/cloud-db:\<TAG\>.  
5. **Image pull details**  
   * If the registry is private, the imagePullSecrets entry must match the **customer’s Docker registry secret name**.  
   * If they use a public registry or anonymous access, this can be dropped.  
6. **Migration job naming & versioning**  
   * The Job name (cloud-db-migration-0-1-17 in our example) encodes a migration version.  
   * For new releases, **architects should bump both**:  
     * The Job name (e.g., cloud-db-migration-0-1-18) and  
     * The image tag (e.g., v0.1.18),  
        so operators can see which migration ran.

### **How to apply this component**

First if the following extensions are not setup:

| \-c 'CREATE EXTENSION IF NOT EXISTS btree\_gin;' \\\-c 'CREATE EXTENSION IF NOT EXISTS pg\_trgm;' |
| :---- |

Run [create-postgres-extensions.sh](http://create-postgres-extensions.sh) while pointing to the cluster:

| \~/go/src/cloud-db/deploy🌴git:(sa-enablement-distroless) ⌚ 21:40:53 » ./create-postgres-extensions.sh📦 Using database name: forge⏳  Waiting for StatefulSet postgres/forge-pg-cluster replicas to be Ready...✅  Postgres StatefulSet is Ready (3/3)🔑  Running extension SQL inside pod forge-pg-cluster-0Defaulted container "postgres" out of: postgres, postgres-exporterCREATE EXTENSIONCREATE EXTENSION✅  Postgres extensions ensured. |
| :---- |

Once the customer-specific fields are updated in the cloud-api Kustomize base (image, registry auth, config, DB secret name, service type), the architect applies it with:

| kubectl apply \-k deploy/kustomize/base  |
| :---- |

## **7.5 cloud‑workflow (namespace cloud-workflow)**

Branch with These changes \-\> [link](https://gitlab-master.nvidia.com/nvmetal/cloud-workflow/-/tree/sa-enablement/deploy/kustomize?ref_type=heads)

### **Role**

cloud-workflow provides the **cloud-side Temporal workers**. It has two deployments:

* cloud-worker → processes workflows in the **cloud** Temporal namespace & queue.  
* site-worker → processes workflows in the **site** Temporal namespace & queue.

Both read a shared config file and differ only in TEMPORAL\_NAMESPACE / TEMPORAL\_QUEUE.

### **7.5.1 Dependencies**

Before cloud-workflow is applied, the following must already be in place:

1. **PostgreSQL \+ schema**  
   * Postgres endpoint reachable (same DB that cloud-db migrates).  
   * DB created (e.g., forge or cloud).  
   * cloud-db migration Job has succeeded.  
   * Credential Secret (in our reference: forge.forge-pg-cluster.credentials) with username / password.  
2. **Temporal**  
   * Frontend gRPC endpoint reachable from this namespace (reference: temporal-frontend-headless.temporal.svc.cluster.local:7233).  
   * Temporal namespaces cloud and site are registered.  
3. **Temporal client TLS Secret**  
   * Secret temporal-client-cloud-certs containing tls.crt, tls.key, ca.crt.  
   * Mounted at /var/secrets/temporal/certs/client-cloud/.  
4. **Image registry**  
   * A registry controlled by the customer where they will **build and push** the cloud-workflow image.

### **7.5.2 Behavior in our reference environment**

* **Namespace:** cloud-workflow  
* **Workload type:**  
  * Deployment cloud-worker (3 replicas).  
  * Deployment site-worker (3 replicas).  
* **Container port:** 8899 (HTTP health/ready; metrics served on 9360).  
* **Liveness / Readiness probes:**  
  * GET /healthz on 8899  
  * GET /readyz on 8899  
* **Config file:** /etc/cloud-workflow/config.yaml  
  * CONFIG\_FILE\_PATH env is set to this path.  
* **Volumes (both deployments):**  
  * forge-pg-cluster-auth → Secret forge.forge-pg-cluster.credentials → /var/secrets/db/auth/  
  * cloud-workflow-config → ConfigMap cloud-workflow-config → /etc/cloud-workflow/  
  * temporal-client-cloud-certs → Secret temporal-client-cloud-certs → /var/secrets/temporal/certs/client-cloud/  
* **Temporal env per deployment:**  
  * cloud-worker: TEMPORAL\_NAMESPACE=cloud, TEMPORAL\_QUEUE=cloud  
  * site-worker:  TEMPORAL\_NAMESPACE=site,  TEMPORAL\_QUEUE=site

#### Internal image we validated with (for reference only):

| nvcr.io/nvidian/nvforge-devel/cloud-workflow:v0.2.33 |
| :---- |

This NVCR image is **only** what we ran internally. Customers **must build their own image from source and push it to their registry**, then update the deployment to use that registry URL.

#### 7.5.3 Kustomize layout (sa-enablement branch)

In the cloud-workflow repo’s sa-enablement branch, we expect:

| deploy/kustomize/base/cloud-workflow/  secrets/    temporal-client-cloud-certs.yaml  \# placeholder TLS secret, empty values  configmap.yaml                      \# cloud-workflow-config (config.yaml)  deployment.yaml                     \# cloud-worker \+ site-worker  service.yaml                        \# Services for each worker deployment  imagepullsecret.yaml  namespace.yaml  kustomization.yaml |
| :---- |

#### 7.5.4 Config (what SAs must change)

**File:** deploy/kustomize/base/cloud-workflow/configmap.yaml

**Key:** data key config.yaml inside the cloud-workflow-config ConfigMap.

Fields that are **customer-specific**:

* log.level → e.g., debug, info, warn.  
* db.\*:  
  * db.host → customer’s Postgres host/FQDN.  
  * db.port → customer’s port (usually 5432).  
  * db.name → name of customer’s DB.  
  * db.user → DB user name.  
  * db.passwordPath → path under /var/secrets/db/auth/ that the app expects.

* temporal.\*:  
  * temporal.host → Temporal frontend DNS name (e.g., temporal-frontend-headless.temporal.svc.cluster.local or the customer’s host).  
  * temporal.port → typically 7233\.  
  * temporal.serverName → server name used for TLS validation (if applicable).  
  * temporal.namespace → should match the worker’s TEMPORAL\_NAMESPACE env.  
  * temporal.queue → should match the worker’s TEMPORAL\_QUEUE env.  
  * temporal.tls.certPath, keyPath, caPath → must align with where temporal-client-cloud-certs is mounted.  
* siteManager.svcEndpoint → full URL for the site API (e.g., https://sitemgr.cloud-site-manager:8100/v1/site).

#### 7.5.5 Secrets (placeholders vs real values)

**File:** deploy/kustomize/base/cloud-workflow/secrets/temporal-client-cloud-certs.yaml

* Contains empty tls.crt, tls.key, ca.crt. This is a placeholder and can be applied as is\!

**DB credentials**

* forge.forge-pg-cluster.credentials must exist in the cluster (ESO or manual).  
* If the customer uses a different secret, they must update the volumes\[\].secret.secretName in deployment.yaml accordingly.

#### 7.5.6 Image and registry (critical change vs internal NVCR)

In the repo’s deployment.yaml, the containers currently reference:

| image: nvcr.io/nvidian/nvforge-devel/cloud-workflow:v0.2.33 |
| :---- |

For customer deployments:

1. **Build from source** (in the cloud-workflow repo):

| docker build \-t \<CUSTOM\_REGISTRY\>/\<PROJECT\>/cloud-workflow:\<TAG\> .docker push \<CUSTOM\_REGISTRY\>/\<PROJECT\>/cloud-workflow:\<TAG\> |
| :---- |

2. **Update the image reference** in the manifests to use **their registry URL**, not NVCR:  
   * Either via **Kustomize images** in deploy/kustomize/base/cloud-workflow/kustomization.yaml:

| images:  \- name: cloud-workflow          \# logical name in deployment.yaml    newName: \<CUSTOM\_REGISTRY\>/\<PROJECT\>/cloud-workflow    newTag: \<TAG\> |
| :---- |

   * Or by editing deployment.yaml directly if you are not using the images: transform.  
3. Ensure imagePullSecrets points at the customer’s registry secret when the registry is private.

#### 7.5.7 Services

The repo defines **internal Services** for each worker:

* cloud-worker Service:  
  * Port 8899 → HTTP/health  
  * Port 9360 → metrics  
* site-worker Service:  
  * Same ports as above.

These are intended for **internal use** (probes/metrics scraping), not direct external traffic.

---

#### 7.5.8 How to apply cloud-workflow

After:

* cloud-db has completed its migrations,  
* DB credentials exist in the expected Secret,  
* Temporal is reachable and cloud / site namespaces exist,  
* temporal-client-cloud-certs and any OTEL secrets are populated appropriately,

the solution architect can deploy cloud-workflow with:

| kubectl apply \-k deploy/kustomize |
| :---- |

This will create:

* Namespace cloud-workflow,  
* ConfigMap cloud-workflow-config,  
* Secrets in cloud-workflow/secrets (as provided/overridden),  
* Deployments cloud-worker and site-worker,  
* Services for cloud-worker and site-worker.

## **7.6 cloud‑api (namespace cloud-api)**

Branch with These changes \-\> [link](https://gitlab-master.nvidia.com/nvmetal/cloud-api/-/merge_requests/1181)

#### Role

cloud-api is the **front-end API** for cloud-side operations. It exposes HTTP endpoints and (optionally) metrics; it reads configuration from a YAML file, connects to the cloud DB, and integrates with Temporal(and grpc backend through temporal),  site-manager, and optional telemetry backends.

#### Reference behavior we run internally

* **Namespace:** cloud-api  
* **Workload type:** Deployment (1 replica per node)  
* **Image / version (reference only):**  
   nvcr.io/nvidian/nvforge-devel/cloud-api:v0.2.76  
* **Port / probes:**  
  * Container port **8388** (named api).  
  * Liveness probe: GET /healthz on 8388\.  
  * Readiness probe: GET /readyz on 8388\.  
* **Service:** LoadBalancer with:  
  * Port **80** → targetPort **8388** (http)  
  * Port **9360** → targetPort **9360** (metrics)  
* **Config file:**  
  * CONFIG\_FILE\_PATH=/etc/cloud-api/config.yaml  
  * Volume cloud-api-config mounts that config at /etc/cloud-api/config.yaml.  
* **Secrets / volumes expected by the deployment:**  
  * forge.forge-pg-cluster.credentials → mounted at /var/secrets/db/auth/  
     (DB username/password)  
  * temporal-client-cloud-certs → mounted at /var/secrets/temporal/certs/client-cloud/  
     (Temporal client TLS: tls.crt, tls.key, ca.crt)

#### Kustomize layout (sa-enablement branch)

Under the cloud-api repo’s sa-enablement [branch](https://gitlab-master.nvidia.com/nvmetal/cloud-api/-/merge_requests/1181):

| deploy/kustomize/base/  secrets/    ssa-client-secret.yaml              \# placeholder    temporal-client-cloud-certs.yaml    \# placeholder  configmap.yaml                        \# cloud-api-config (config.yaml)  Deployment.yaml   Imagepullsecret.yaml                  \# replace the secret value  service.yaml  namespace.yaml  kustomization.yaml |
| :---- |

### **Config & integration details**

#### 1\. Application config (config.yaml)

This is the **main file SAs need to customize** for each customer.

* **File:** deploy/kustomize/base/cloud-api/configmap.yaml  
* **Key:** cloud-api-config.data\[\`config.yaml\`\]  
* **Important fields inside config.yaml:**  
  * log.level → log level (e.g., debug, info).  
  * kas.legacyJwksUrl, kas.ssaJwksUrl, kas.starfleetJwksUrl → JWKS endpoints for various IdPs / SSA integration (customer URLs).  
  * db.host, db.port, db.name, db.user, db.passwordPath → DB connection:  
    * host and port must match **customer’s Postgres service** (often the same as cloud-db-config values).  
    * name and user must correspond to their DB and role.  
    * passwordPath should match the path under /var/secrets/db/auth/ where the password is mounted.  
  * temporal.host, temporal.port, temporal.serverName, temporal.namespace, temporal.queue:  
    * host must be the Temporal frontend or headless service for Temporal (e.g., temporal-frontend-headless.temporal.svc.cluster.local).  
    * namespace/queue must match the **Temporal namespace / task queue** used by cloud workers (e.g., **cloud**).  
    * tls.certPath, keyPath, caPath must match how temporal-client-cloud-certs is mounted.  
  * siteManager.enabled, siteManager.svcEndpoint → URL for site-manager’s v1/site endpoint.

#### Customer actions (config.yaml):

* Update all **URLs**, **hostnames**, and **ports** to match the customer’s auth, Temporal, DB, site-manager, and ZincSearch endpoints.  
* Update db.\* to align with their Postgres configuration.  
* Confirm TLS paths line up with the volume mount paths in deployment.yaml.

#### 2\. Deployment & container image

* **File:** deploy/kustomize/base/cloud-api/**deployment.yaml**  
* **Key elements:**  
  * spec.replicas: default **2**; tune per customer.  
  * Container:  
    * Name: cloud-api  
    * Image (reference): nvcr.io/nvidian/nvforge-devel/cloud-api:v0.2.76  
    * Env: CONFIG\_FILE\_PATH=/etc/cloud-api/config.yaml  
    * Ports: 8388 named api  
  * Volume mounts:  
    * /var/secrets/db/auth/ → forge.forge-pg-cluster.credentials  
    * /var/secrets/ssa/ → ssa-client-secret  
    * /etc/cloud-api/ → cloud-api-config  
    * /var/secrets/temporal/certs/client-cloud/ → temporal-client-cloud-certs

**Customer actions (deployment):**

* Replace the image with their own build:  
  * Build: docker build \-t \<REGISTRY\>/\<PROJECT\>/cloud-api:\<TAG\> .  
  * Push:  docker push \<REGISTRY\>/\<PROJECT\>/cloud-api:\<TAG\>  
* Update the Kustomize images mapping:  
  * **File:** deploy/kustomize/base/cloud-api/kustomization.yaml  
     images: → set newName: \<REGISTRY\>/\<PROJECT\>/cloud-api, newTag: \<TAG\>.  
* Decide if they want imagePullSecrets:  
  * For private registries, ensure:

| imagePullSecrets:  \- name: \<CUSTOM\_IMAGEPULLSECRET\> |
| :---- |

#### 3\. Database credentials Secret

* The deployment mounts a secret called forge.forge-pg-cluster.credentials at /var/secrets/db/auth/.  
* That secret is expected to have **username** and **password** keys.

**Customer actions (DB Secret):**

* Either:  
  * Keep the name forge.forge-pg-cluster.credentials and create/populate it (ideally via ESO), **or**  
  * Change volumes\[\].secret.secretName \+ any env.valueFrom.secretKeyRef.name in deployment.yaml to their Secret name (e.g., cloud-db-credentials).

**File:** deploy/kustomize/base/cloud-api/deployment.yaml

#### 4\. SSA client secret (ssa-client-secret)

There are two related files:

* Placeholder Secret with empty value (original pattern)  
   **File:** deploy/kustomize/base/cloud-api/secrets/ssa-client-secret.yaml  
  * client-secret: "" (base64 of empty) — **safe for non-SSA environments**.  
* Secret with placeholder value c3NhLXJlcGxhY2U= (base64 for "ssa-replace")  
   **File:** deploy/kustomize/base/cloud-api/ssa-placeholder-secret.yaml  
  * This is **only a placeholder**; customers must override it with:  
    * A real SSA client secret (if using SSA), or  
    * An empty value in non-SSA deployments.

**Customer actions (SSA Secret):**

* If SSA is in scope:  
  * Set client-secret to the **base64-encoded** SSA client secret.  
* If SSA is **not** used:  
  * Keep the placeholder

#### 5\. Temporal client TLS certs (temporal-client-cloud-certs)

* Placeholder TLS Secret with keys tls.crt, tls.key, ca.crt set to "" (empty).  
   **File:** deploy/kustomize/base/cloud-api/secrets/temporal-client-cloud-certs.yaml  
* Mounted at /var/secrets/temporal/certs/client-cloud/.

**Customer actions (Temporal TLS):**

* If Temporal requires mTLS (our reference assumes yes):  
  * Fill in **base64-encoded** tls.crt, tls.key, and ca.crt for the client cert, key, and CA.  
* If they run Temporal without TLS:  
  * They can leave the cert values blank but must ensure config.yaml TLS paths and Temporal deployment tolerate non-TLS (or remove TLS config from config.yaml).

#### 6\. Service exposure

* **File:** deploy/kustomize/base/cloud-api/service.yaml  
* **Behavior in reference:**  
  * Type LoadBalancer.  
  * Port **80/TCP** → targetPort: 8388 (HTTP API).  
  * Port **9360/TCP** → targetPort: 9360 (metrics).  
* Selector:  
  * app: cloud-api (matches labels in deployment.yaml).

**Customer actions (Service):**

* Decide **how** cloud-api should be exposed:  
  * **Internal only**: set type to ClusterIP and front with Ingress/HTTPProxy.  
  * **External**: keep or adjust LoadBalancer and connect via their LB/Ingress.  
* Adjust ports if they need different external port numbers, but keep targetPort aligned with the container’s 8388 for API and 9360 for metrics.

---

**How to apply cloud-api**

Once all customer-specific knobs are updated:

* configmap.yaml (app config),  
* deployment.yaml (image, registry, secrets, volumes),  
* service.yaml (type/ports),  
* secrets in secrets/ and ssa-placeholder-secret.yaml (as needed),  
* and otel.env (if telemetry is used),

the architect can deploy cloud-api by running:

| kubectl apply \-k deploy/kustomize/base/cloud-api |
| :---- |

https://docs.google.com/document/d/174MUDmsOAtZiE7qrYQ7D-cJFQR40B83xeBHJFT\_ivkM/edit?tab=t.0

## **7.7 cloud‑site‑manager (namespace cloud-site-manager)**

#### Role

cloud-site-manager (a.k.a. *sitemgr*) is “site registry” service. It:

* Owns the Site CRD and stores per‑site metadata as Site objects.  
* Talks to **credsmgr** to mint per‑site credentials / CA material.  
* Exposes an HTTPS API on port **8100** that the bootstrap flow calls to:  
  * create a new site (/v1/site), and  
  * later allow the site agent to fetch its One TIme Password(OTP) credentials.

---

### **7.7.1 Dependencies**

Before you deploy cloud-site-manager, you should already have:

* **cert-manager \+ cloud-cert-manager (credsmgr)**:  
  * vault-issuer ClusterIssuer configured and working.  
  * credsmgr Service reachable at https://credsmgr.cert-manager:8000 (or your equivalent).  
* **Vault root CA secrets**:  
  * vault-root-ca-certificate and vault-root-ca-private-key in the cert-manager namespace (created via the helper script in 7.0 or by your own process).  
* **Kubernetes CRD support** (standard; no extra setup).

There is **no direct Postgres or Temporal dependency** in this manifest; those get exercised later by other components through the site bootstrap workflows.

### **7.7.2 Behavior in our reference environment**

**Namespace**: cloud-site-manager

**Workloads**:

* **CRD**: sites.forge.nvidia.io  
  * Kind: Site  
  * Scope: Namespaced  
  * spec and status are both “free‑form” (x-kubernetes-preserve-unknown-fields: true), so the CRD doesn’t constrain your schema.  
* **Deployment**: csm  
  * Replicas: **3**  
  * Container: sitemgr  
  * Image (reference only):  
     nvcr.io/nvidian/nvforge-devel/cloud-site-manager:v0.1.16  
  * Arguments:  
    * \--listen-port=8100  
    * \--creds-manager-url=https://credsmgr.cert-manager:8000  
    * \--ingress-host=sitemgr.cloud-site-manager.svc.cluster.local  
  * Environment:  
    * SITE\_MANAGER\_NS from metadata.namespace  
    * envFrom: secretRef: otel-lightstep (optional telemetry; can be stubbed or removed)  
  * Probes:  
    * Liveness / Readiness / Startup: HTTPS GET /healthz on port 8100  
  * ServiceAccount: site-manager  
* **RBAC** (namespace‑scoped):  
  * Role site-manager grants full access to sites and sites/status in API group forge.nvidia.io.  
  * RoleBinding site-manager binds the Role to the site-manager ServiceAccount in cloud-site-manager.  
* **Service**: sitemgr  
  * Type: ClusterIP  
  * Port 8100 → targetPort 8100  
  * Selector: app.kubernetes.io/name: csm  
  * This is what your bootstrap flow curls:  
     https://sitemgr.cloud-site-manager:8100/v1/site

---

### **7.7.3 Kustomize layout (sa‑enablement branch)**

In the cloud-site-manager repo (sa‑enablement branch), the base Kustomize tree for this component looks like:

* deploy/kustomize/  
  * namespace.yaml              – defines cloud-site-manager namespace  
  * crd-site.yaml               – Site CRD (sites.forge.nvidia.io)  
  * deployment.yaml             – csm Deployment (sitemgr container)  
  * serviceaccount.yaml         – site-manager ServiceAccount  
  * serviceaccount-role.yaml    – Role for CRD access  
  * serviceaccount-rolebinding.yaml – binds Role to site-manager SA  
  * sitemgr-service.yaml        – ClusterIP Service on port 8100  
  * kustomization.yaml          – includes the above and overrides the image

kustomization.yaml sets the **reference** image to:

| images:  \- name: sitemgr    newName: nvcr.io/nvidian/nvforge-devel/cloud-site-manager    newTag: v0.1.16 |
| :---- |

### 

### **7.7.4 Customer knobs (what SAs must adjust)**

1. **Image & registry**

Customers must build cloud-site-manager from source and push to their own registry, then update the image mapping in kustomization.yaml.

Example:

| \# Build and pushdocker build \-t \<REGISTRY\>/\<PROJECT\>/cloud-site-manager:\<TAG\> .docker push \<REGISTRY\>/\<PROJECT\>/cloud-site-manager:\<TAG\> |
| :---- |

Then in the Kustomize base:

| images:  \- name: sitemgr    newName: \<REGISTRY\>/\<PROJECT\>/cloud-site-manager    newTag: \<TAG\> |
| :---- |

If their registry is private, ensure an appropriate imagePullSecrets entry (and corresponding Secret) is present in the Deployment; otherwise it can be removed.

2. **credsmgr URL**

The Deployment passes:

| \--creds-manager-url=https://credsmgr.cert-manager:8000 |
| :---- |

If the customer:

* Renames the credsmgr Service,  
* Runs it in a different namespace,

they **must update this flag** to point at their actual URL. Whatever CA service this URL references must:

* Trust the same root CA that site agents will trust, and  
* Be able to issue the per‑site certs/OTP material your bootstrap flow expects.

---

3. **Ingress / internal host**

The \--ingress-host flag is currently:

| \--ingress-host=sitemgr.cloud-site-manager.svc.cluster.local |
| :---- |

This is the hostname sitemgr uses when constructing URLs and, in some flows, may be placed into certificates or responses.

Customer options:

* Keep it as the internal DNS name if the service will only ever be called in‑cluster.  
* Change it to an internal or external FQDN (e.g. sitemgr.\<customer-domain\>), but then:  
  * Make sure their DNS / ingress routes that hostname to the sitemgr Service.  
  * Ensure any TLS certs used for sitemgr include this hostname in their SANs.

### **7.7.5 How to apply cloud-site-manager**

After:

* cloud-cert-manager / credsmgr is running and reachable at the URL you specified,  
* The Vault root CA secrets (vault-root-ca-certificate, vault-root-ca-private-key) exist in cert-manager,  
* The image reference has been updated to the customer’s registry,  
* Any telemetry Secret decisions are made,

the SA can deploy cloud-site-manager with:

| $ kubectl apply \-k deploy/kustomize namespace/cloud-site-manager created customresourcedefinition.apiextensions.k8s.io/sites.forge.nvidia.io created serviceaccount/site-manager created role.rbac.authorization.k8s.io/site-manager created rolebinding.rbac.authorization.k8s.io/site-manager created service/sitemgr created deployment.apps/csm created  |
| :---- |

This creates:

* Namespace cloud-site-manager,  
* The Site CRD in the cluster,  
* ServiceAccount \+ Role \+ RoleBinding for site-manager,  
* The csm Deployment (sitemgr) and sitemgr Service.

## **7.8 Carbide Core**

The following sections are detailed documentation of each carbide component.  
For SA enablement and deployment commands for dev8 please jump to section 7.13 SA Enablement \- Carbide bring up

MAKE SURE ALL IS APPLIED IN forge-system Name Space

### **7.8.1 Carbide API (Control Plane API)**

Path in carbide directory:  deploy/carbide-base/api  
It corresponds to the following directory:

| deploy/carbide-base/api/├── api-service.yaml├── certificate.yaml├── config-files│   ├── carbide-api-config.toml│   └── casbin-policy.csv├── deployment.yaml├── kustomization.yaml├── metrics-service.yaml├── migration.yaml├── profiler-service.yaml├── README.md├── role.yaml├── rolebinding.yaml└── service-account.yaml |
| :---- |

#### High-Level Overview

* **Primary workload:** Deployment/carbide-api  
* **Database migration hook:** Job/carbide-api-migrate  
* **Services:**  
  * Service/carbide-api (gRPC, port 1079\)  
  * Service/carbide-api-metrics (metrics, port 1080\)  
  * Service/carbide-api-profiler (profiler, port 1081\)  
* **Config:**  
  * ConfigMap/carbide-api-config-files (generated from config-files/)  
  * ConfigMap/carbide-api-site-config-files (site-specific, empty at base)  
* **TLS & identity:**  
  * Certificate/carbide-api-certificate → Secret/carbide-api-certificate  
* **RBAC:**  
  * ServiceAccount/carbide-api  
  * Role/carbide-api  
  * RoleBinding/carbide-api

#### **External dependencies** you must provide:

* Secrets:  
  * Carbide-vault-token (Vault’s root\_token)  
  * carbide-vault-approle-tokens  
  * forge-system.carbide.forge-pg-cluster.credentials  
  * forge-roots  
* ConfigMaps:  
  * vault-cluster-info  
  * forge-system-carbide-database-config

#### Kustomization Entrypoint (kustomization.yaml)

**kustomization.yaml** is the entrypoint that wires all manifests together and applies common labels.

### **Behavior**

* Adds labels to all resources and their selectors/templates:  
  * app.kubernetes.io/name: carbide-api  
  * app.kubernetes.io/component: api

* Declares the resources:  
  * api-service.yaml  
  * certificate.yaml  
  * deployment.yaml  
  * metrics-service.yaml  
  * profiler-service.yaml  
  * role.yaml  
  * rolebinding.yaml  
  * service-account.yaml  
  * migration.yaml  
* Generates config maps:  
  * **ConfigMap/carbide-api-config-files**  
    * From:  
      * config-files/carbide-api-config.toml (described below)  
      * config-files/casbin-policy.csv (described below)  
    * Mounted into the deployment at /etc/forge/carbide-api.

  * **ConfigMap/carbide-api-site-config-files**  
    * files: \[\] in base.  
    * Overlays are expected to add site-specific config files  
      (e.g. carbide-api-site-config.toml).  
    * Mounted into the deployment at /etc/forge/carbide-api/site.

* generatorOptions.disableNameSuffixHash: true  
  To keep mounts and paths stable  
  * Ensures config map names stay exactly:  
    * carbide-api-config-files  
    * carbide-api-site-config-files

###  **Configuration Files (config-files/)**

| config-files/carbide-api-config.toml |
| :---- |

Shared runtime configuration for carbide-api. Environment-specific overrides should go in a carbide-api-site-config.toml in an overlay.

**Current keys (with default values):**

* **Core endpoints**  
  * **listen \= "\[::\]:1079"**   
  * metrics\_endpoint \= "\[::\]:1080"   
  * profiler\_endpoint \= "\[::\]:1081"   
* **Database**  
  * **database\_url \= "postgres://replaced-by-env-var"**  
    * **Important:** This is overridden at runtime by the CARBIDE\_API\_DATABASE\_URL environment variable in deployment.yaml.  
* **Firmware / DPU**  
  * **asn \= 4294967000**   
  * initial\_dpu\_agent\_upgrade\_policy \= "up\_only"   
  * dpu\_nic\_firmware\_intial\_update\_enabled \= false   
  * dpu\_nic\_firmware\_reprovision\_update\_enabled \= true   
  * dpu\_nic\_firmware\_update\_version \= { "BlueField SoC" \= "24.42.1000", "BlueField-3 SmartNIC Main Card" \= "32.42.1000" } 

  * max\_concurrent\_machine\_updates \= 10   
  * nvue\_enabled \= true   
  * dhcp\_servers \= \[\] (expected to be filled per site via site config)   
  * route\_servers \= \[\] (for FNN; currently not used)   
  * attestation\_enabled \= true   
  * bypass\_rbac \= true   
    MAKE SURE WE BYPASS RBAC since we’re not using nvint certs

* **Host health**

| \[host\_health\]hardware\_health\_reports \= "MonitorOnly" |
| :---- |

* **Site explorer**

| \[site\_explorer\]enabled \= truecreate\_machines \= true |
| :---- |

* **Firmware global**

| \[firmware\_global\]autoupdate \= true |
| :---- |

* **TLS**

| \[tls\]identity\_pemfile\_path \= "/var/run/secrets/spiffe.io/tls.crt"identity\_keyfile\_path \= "/var/run/secrets/spiffe.io/tls.key"root\_cafile\_path \= "/var/run/secrets/spiffe.io/ca.crt"admin\_root\_cafile\_path \= "/etc/forge/carbide-api/site/admin\_root\_cert\_pem" |
| :---- |

* **Auth**

| \[auth\]permissive\_mode \= falsecasbin\_policy\_file \= "/etc/forge/carbide-api/casbin-policy.csv" |
| :---- |

* **Machine state controller**

| \[machine\_state\_controller\]failure\_retry\_time \= "45m" |
| :---- |

* **Network security group**

| \[network\_security\_group\]stateful\_acls\_enabled \= falsemax\_network\_security\_group\_size \= 200 |
| :---- |

* **Machine validation**

| \[machine\_validation\_config\]enabled \= truetest\_selection\_mode \= "EnableAll" |
| :---- |

**How to customize:**

* Create a **site-specific config** file (e.g. carbide-api-site-config.toml) in your overlay and add it to carbide-api-site-config-files via kustomize.

* Override keys that are environment dependent   
  * Dhcp\_servers  
  * Route\_servers  
  * Admin\_root\_cafile\_path  
  * Etc…

#### Casbin-policy Configuration

| config-files/casbin-policy.csv |
| :---- |

Casbin RBAC policy for carbide-api.

**Key points:**

* g rules bind principals (SPIFFE IDs or machine IDs) to roles.  
* p rules grant actions (Forge RPC methods) to roles or principals.  
* Example existing rules:  
  * Map SPIFFE IDs to roles:  
    * g, spiffe-service-id/carbide-dhcp, carbide-dhcp  
    * g, spiffe-service-id/carbide-dns, carbide-dns  
    * g, spiffe-machine-id, machine  
  * Allow carbide-dhcp to call forge/DiscoverDhcp.  
  * Allow carbide-dns to call forge/LookupRecord.  
  * p, trusted-certificate, forge/\* — broad access for trusted certificates (intended to be tightened over time).  
  * p, anonymous, forge/Version, forge/DiscoverMachine, forge/ReportForgeScoutError, forge/AttestQuote, and multiple machine-related methods.

**How to customize:**

* Update or add g and p rules as new services/methods are introduced.  
* Carefully tighten or replace the trusted-certificate, forge/\* rule as enforcement becomes stricter.

####  Deployment (deployment.yaml)

### **4.1 Basic spec**

* **Kind:** Deployment  
* **Name:** carbide-api  
* **Namespace:** default  
* **Replicas:** 1  
* **Strategy:** RollingUpdate  
* **Selector:** app.kubernetes.io/name: carbide-api  
* **Pod annotations:**  
  * kubectl.kubernetes.io/default-container: carbide-api

### **Container**

* **Name:** carbide-api  
* **Image:** nvcr.io/nvidian/nvforge/nvmetal-carbide:latest  
  * Typically pinned via overlay in real deployments.  
* **Command:**

| /opt/carbide/carbide-api run \\  \--config-path /etc/forge/carbide-api/carbide-api-config.toml \\  \--site-config-path /etc/forge/carbide-api/site/carbide-api-site-config.toml |
| :---- |

* **Security context:**  
  * Adds capability SYS\_PTRACE.  
* **Ports:**  
  * **grpc \- containerPort: 1079**   
  * metrics \- containerPort: 1080   
  * profiler \- containerPort: 1081   
* **Health checks:**  
  * livenessProbe:  
    * HTTP GET on port 1080 (metrics port).  
    * initialDelaySeconds: 20   
    * periodSeconds: 20   
    * timeoutSeconds: 20   
    * failureThreshold: 3   
* **ServiceAccount:**  
  * serviceAccountName: carbide-api

**Volume mounts & volumes**

**Pod volumes:**

1. carbide-api-config-files  
   * Type: ConfigMap  
   * Name: carbide-api-config-files  
   * **Mounted at:** /etc/forge/carbide-api

2. carbide-api-site-config-files  
   * Type: ConfigMap  
   * Name: carbide-api-site-config-files  
   * **Mounted at:** /etc/forge/carbide-api/site

3. persistence  
   * Type: emptyDir: {}  
   * **Mounted at:** /mnt/persistence  
4. spiffe  
   * Type: Secret  
   * secretName: carbide-api-certificate  
   * **Mounted at:** /var/run/secrets/spiffe.io (readOnly)

5. forge-roots  
   * Type: Secret  
   * secretName: forge-roots  
   * **Mounted at:** /var/run/secrets/forge-roots (readOnly)

6. firmware  
   * Type: emptyDir: {}  
   * **Mounted at:** /opt/carbide/firmware (readOnly in container)

**Deployment Environment variables**

Below is a reference of all env vars in the main carbide-api container.

##### Vault integration

* VAULT\_ROLE\_ID  
  * **Source:** Secret/carbide-vault-approle-tokens, key VAULT\_ROLE\_ID  
  * **Optional:** true  
  * **Purpose:** AppRole auth to Vault (if used).

* VAULT\_SECRET\_ID  
  * **Source:** Secret/carbide-vault-approle-tokens, key VAULT\_SECRET\_ID  
  * **Optional:** true  
  * **Purpose:** AppRole secret part.

* VAULT\_TOKEN  
  * **Source:** Secret/carbide-vault-token, key token  
  * **Purpose:** Direct Vault token used by the service.

* VAULT\_ADDR  
  * **Source:** ConfigMap/vault-cluster-info, key VAULT\_SERVICE  
  * **Value example:** e.g. https://vault.forge.svc.cluster.local:8200 (cluster-specific)  
  * **Purpose:** Vault base URL.

* VAULT\_MOUNT\_LOCATION

* VAULT\_KV\_MOUNT\_LOCATION  
  * **Source:** ConfigMap/vault-cluster-info, key FORGE\_VAULT\_MOUNT  
  * **Purpose:** Path where Forge secrets are mounted

* VAULT\_PKI\_MOUNT\_LOCATION  
  * **Source:** ConfigMap/vault-cluster-info, key FORGE\_VAULT\_PKI\_MOUNT  
  * **Purpose:** Path for Vault PKI backend

* VAULT\_PKI\_ROLE\_NAME  
  * **Value (current):** "forge-cluster"  
  * **Purpose:** PKI role used for certificate issuance.

##### Database configuration

These are **shared** between the deployment and the migration job.

* DATASTORE\_USER  
  * **Source:** Secret/forge-system.carbide.forge-pg-cluster.credentials, key username  
  * **Purpose:** PostgreSQL username.

* DATASTORE\_PASSWORD  
  * **Source:** Secret/forge-system.carbide.forge-pg-cluster.credentials, key password  
  * **Purpose:** PostgreSQL password.

* DATASTORE\_HOST  
  * **Source:** ConfigMap/forge-system-carbide-database-config, key DB\_HOST  
  * **Purpose:** PostgreSQL host (e.g. forge-pg-cluster.default.svc).

* DATASTORE\_PORT  
  * **Source:** ConfigMap/forge-system-carbide-database-config, key DB\_PORT  
  * **Purpose:** PostgreSQL port (e.g. 5432).

* DATASTORE\_NAME  
  * **Source:** ConfigMap/forge-system-carbide-database-config, key DB\_NAME  
  * **Purpose:** PostgreSQL database name.

* CARBIDE\_API\_DATABASE\_URL  
  * **Value (computed):**

| postgres://$(DATASTORE\_USER):$(DATASTORE\_PASSWORD)@$(DATASTORE\_HOST):$(DATASTORE\_PORT)/$(DATASTORE\_NAME) |
| :---- |

  * **Purpose:** Final database URL that overrides database\_url in carbide-api-config.toml.

#### **Web / auth**

* CARBIDE\_WEB\_PRIVATE\_COOKIEJAR\_KEY  
  * **Value (current):** $(DATASTORE\_PASSWORD) (i.e. the DB password)  
  * **Purpose:** Key for encrypting cookie jar data.  
  * **Note:** In a hardened environment, you may want a dedicated secret instead of reusing the DB password.

* **Commented-out env vars (placeholders for SSO integration)**:  
  * CARBIDE\_WEB\_OAUTH2\_CLIENT\_SECRET   
    * Would come from Secret/azure-sso-carbide-web-client-secret, key client\_secret.

  * CARBIDE\_WEB\_HOSTNAME   
    * Would come from ConfigMap/carbide-web-api-hostname, key hostname.

  * CARBIDE\_WEB\_AUTH\_TYPE   
    * Would be set to oauth2 for SSO.

##### Debugging / runtime

* BITNAMI\_DEBUG  
  * **Value:** "false"  
  * **Purpose:** Standard debug toggle for Bitnami images; likely unused but set to false.

* RUST\_BACKTRACE  
  * **Value:** "full"  
  * **Purpose:** Enables full Rust stack traces on panic.

* RUST\_LIB\_BACKTRACE  
  * **Value:** "0"  
  * **Purpose:** Controls backtrace behavior for Rust libraries (see Rust docs).

#### Migration Job (migration.yaml)

Runs database migrations for the carbide-api database before applying other changes. 

* **Kind:** Job  
* **Name:** carbide-api-migrate  
* **Namespace:** forge-system  
* **Pod spec:**  
  * restartPolicy: OnFailure  
  * Container:  
    * **Name:** carbide-api-migrate  
    * **Image:** nvcr.io/nvidian/nvforge/nvmetal-carbide:latest  
    * **Command:**

| /opt/carbide/carbide-api migrate \\  \--datastore="postgres://${DATASTORE\_USER}:${DATASTORE\_PASSWORD}@${DATASTORE\_HOST}:${DATASTORE\_PORT}/${DATASTORE\_NAME}" |
| :---- |

    * Env vars: same DATASTORE\_\* variables as in the main deployment, with identical sources.

**Configuration impact:**

* Any changes to DB host, port, name, user, or password must be reflected in:  
  * Secret/forge-system.carbide.forge-pg-cluster.credentials   
  * ConfigMap/forge-system-carbide-database-config 

#### Services api-service.yaml

* **Kind:** Service  
* **Name:** carbide-api  
* **Namespace:** forge-system  
* **Labels:**  
  * app: carbide-api  
* **Spec:**  
  * Ports:  
    * port: 1079  
    * name: grpc  
    * protocol: TCP

#### Services metrics-service.yaml

* **Kind:** Service  
* **Name:** carbide-api-metrics  
* **Labels:**  
  * app: carbide-api  
  * app.kubernetes.io/metrics: carbide-api  
* **Spec:**  
  * Ports:  
    * port: 1080  
    * name: http  
    * protocol: TCP  
    * targetPort: 1080

#### Services profiler-service.yaml

* **Kind:** Service  
* **Name:** carbide-api-profiler  
* **Labels:**  
  * app: carbide-api  
  * app.kubernetes.io/metrics: carbide-api  
* **Spec:**  
  * Ports:  
    * port: 1081  
    * name: http  
    * protocol: TCP  
    * targetPort: 1081  
* **Selector:** also absent in base; add via overlay as needed.

#### TLS Certificate (certificate.yaml)

Requests a TLS certificate via cert-manager for the carbide-api service; this is used for SPIFFE-aligned identities.

**Spec**

* **Kind:** Certificate  
* **Name:** carbide-api-certificate  
* **Namespace:** default  
* **spec:**  
  * duration: 720h0m0s (30 days)  
  * renewBefore: 360h0m0s (15 days before expiration)  
  * dnsNames:  
    * carbide-api.default.svc.cluster.local   
  * uris:  
    * spiffe://forge.local/default/sa/carbide-api   
      * **Note:** SPIFFE trust domain "forge.local" is currently hard-coded; should be updated if your environment uses a different domain.  
  * privateKey:  
    * algorithm: ECDSA  
    * size: 384  
  * issuerRef:  
    * kind: ClusterIssuer  
    * Name: vault-forge-issuer   
      (Must have a trusted issuer)  
    * group: cert-manager.io  
  * secretName: carbide-api-certificate

##### Resulting secret

* **Secret name:** carbide-api-certificate  
* **Used by:**  
  * Deployment volume spiffe, mounted at /var/run/secrets/spiffe.io.  
  * carbide-api-config.toml TLS section expects:  
    * /var/run/secrets/spiffe.io/tls.crt  
    * /var/run/secrets/spiffe.io/tls.key  
    * /var/run/secrets/spiffe.io/ca.crt

#### Service Account and RBAC (service-account.yaml)

* **Kind:** ServiceAccount  
* **Name:** carbide-api  
* **Namespace:** default  
* **Used by:**  
  * Deployment/carbide-api via serviceAccountName: carbide-api

#### Service Account and RBAC (role.yaml)

* **Kind:** Role  
* **Name:** carbide-api  
* **Namespace:** default  
* **Rules:**  
  * apiGroups: \["cert-manager.io"\]  
  * resources: \["certificaterequests"\]  
  * verbs: \["create"\]

This allows the service (via cert-manager flows) to create CertificateRequest resources in the namespace.

#### Service Account and RBAC  (rolebinding.yaml)

* **Kind:** RoleBinding  
* **Name:** carbide-api  
* **Namespace:** default (implicitly)  
* **roleRef:**  
  * apiGroup: rbac.authorization.k8s.io  
  * kind: Role  
  * name: carbide-api  
* **subjects:**  
  * Kind: ServiceAccount  
  * Name: carbide-api  
  * Namespace: default

Binds the local Role to the carbide-api service account.

#### Carbide-api Configuration & Dependency Matrix

This section summarizes **what you must provide** and **how it is used**.

#### Secrets

1. **Secret/carbide-vault-token**  
   * Keys:  
     * token – Vault token value.  
   * Used by:  
     * Deployment (env VAULT\_TOKEN).  
   * Purpose:  
     * Authenticate carbide-api to Vault, if not using AppRole.

2. **Secret/carbide-vault-approle-tokens** (optional)  
   * Keys:  
     * VAULT\_ROLE\_ID  
     * VAULT\_SECRET\_ID  
   * Used by:  
     * Deployment (env VAULT\_ROLE\_ID, VAULT\_SECRET\_ID).  
   * Purpose:  
     * AppRole authentication to Vault.

3. **Secret/forge-system.carbide.forge-pg-cluster.credentials**  
   * Keys:  
     * username  
     * password  
   * Used by:  
     * Deployment (DATASTORE\_USER, DATASTORE\_PASSWORD)  
     * Job/carbide-api-migrate (same envs)  
   * Purpose:  
     * PostgreSQL credentials for both application and migrations.

4. **Secret/forge-roots**  
   * Keys:  
     * Implementation-defined (e.g. ca.crt, etc.).  
   * Used by:  
     * Deployment volume mount at /var/run/secrets/forge-roots.  
   * Purpose:  
     * Additional root CA bundle for validating peer certificates.

5. **Secret/carbide-api-certificate**  
   * Created by:  
     * Certificate/carbide-api-certificate (via cert-manager).  
   * Expected keys:  
     * tls.crt, tls.key, ca.crt (typical TLS secret structure).  
   * Used by:  
     * Deployment volume spiffe.  
     * TLS paths in carbide-api-config.toml.

#### ConfigMaps

1. **ConfigMap/vault-cluster-info**  
   * Required keys:  
     * VAULT\_SERVICE → used as VAULT\_ADDR  
     * FORGE\_VAULT\_MOUNT → used as VAULT\_MOUNT\_LOCATION, VAULT\_KV\_MOUNT\_LOCATION  
     * FORGE\_VAULT\_PKI\_MOUNT → used as VAULT\_PKI\_MOUNT\_LOCATION  
   * Used by:  
     * Deployment env vars.  
   * Purpose:  
     * Central configuration for Vault endpoints and mount paths.

2. **ConfigMap/forge-system-carbide-database-config**  
   * Required keys:  
     * DB\_HOST → DATASTORE\_HOST  
     * DB\_PORT → DATASTORE\_PORT  
     * DB\_NAME → DATASTORE\_NAME

   * Used by:  
     * Deployment and Job/carbide-api-migrate.  
   * Purpose:  
     * Central configuration for database endpoint information.

3. **ConfigMap/carbide-api-config-files**  
   * Generated from:  
     * config-files/carbide-api-config.toml  
     * config-files/casbin-policy.csv  
   * Used by:  
     * Deployment (mounted at /etc/forge/carbide-api).

4. **ConfigMap/carbide-api-site-config-files**  
   * Generated with files: \[\] in base; overlays should add site-specific files.  
   * Used by:  
     * Deployment (mounted at /etc/forge/carbide-api/site).

### **7.8.2 Carbide DHCP**

Kubernetes manifests for the carbide-dhcp service and all its configuration surfaces: environment variables, secrets, config maps, TLS certificates, RBAC, and how the components fit together at runtime.

| dhcp/├── certificate.yaml├── deployment.yaml├── kustomization.yaml├── metrics-service.yaml├── role.yaml├── rolebinding.yaml├── service-account.yaml└── service.yaml |
| :---- |

carbide-dhcp is a DHCP service (using Kea DHCPv4) that runs inside Kubernetes and integrates with Forge via mTLS SPIFFE certificates.  
It exposes:

* DHCP traffic on **UDP port 67**  
* Metrics on **TCP port 1089**

**Primary resources:**

* Workload:  
  * Deployment/carbide-dhcp  
* Networking:  
  * Service/carbide-dhcp (DHCP UDP 67\)  
  * Service/carbide-dhcp-metrics (HTTP metrics on 1089\)  
* Identity / TLS:  
  * Certificate/carbide-dhcp-certificate → Secret/carbide-dhcp-certificate  
* Config:  
  * External ConfigMap/carbide-dhcp-config (Kea config JSON)  
* RBAC:  
  * ServiceAccount/carbide-dhcp  
  * Role/carbide-dhcp  
  * RoleBinding/carbide-dhcp  
* Kustomize wiring:  
  * kustomization.yaml

**External dependencies you must provide:**

* ConfigMap/carbide-dhcp-config (Kea DHCP config, including kea\_config.json key or similar)  
* ClusterIssuer/vault-forge-issuer (for cert-manager)  
* cert-manager itself (to reconcile Certificate into Secret)

#### Kustomization Entrypoint (kustomization.yaml)

entrypoint that wires all carbide-dhcp manifests together and applies common labels so everything is consistently addressable.

**Labels:**

Adds labels to resources, selectors, and templates:

* app.kubernetes.io/name: carbide-dhcp  
* app.kubernetes.io/component: dhcp

Both includeSelectors: true and includeTemplates: true are set, so these labels propagate to:

* Pod templates  
* Service selectors (if present)  
* Other labeled fields

**Resources included:**

* certificate.yaml  
* deployment.yaml  
* metrics-service.yaml  
* role.yaml  
* rolebinding.yaml  
* service-account.yaml  
* service.yaml

**The carbide-dhcp-config config map is expected to be defined elsewhere** (e.g., an overlay or another base).

* For instance for SA-Enablement we define it under deploy/kustomization.yaml in carbide repo

#### Deployment (deployment.yaml)

### **3.1 Basic spec**

* **Kind:** Deployment  
* **Name:** carbide-dhcp  
* **Namespace:** forge-system  
* **spec:**  
  * replicas: 1  
  * strategy.type: RollingUpdate  
  * selector.matchLabels:  
    * app.kubernetes.io/name: carbide-dhcp

**Container:**

* **Name:** carbide-dhcp  
* **Image:** nvcr.io/nvidian/nvforge/nvmetal-carbide:latest   
  * Typically overridden with a pinned version in an overlay.  
* **ImagePullPolicy:** IfNotPresent  
* **Command:**

| sh \-c 'mkdir \-vp /var/run/kea /var/lib/kea && /sbin/kea-dhcp4 \-c /tmp/kea\_config.json' |
| :---- |

  * Creates runtime directories for Kea:  
    * /var/run/kea  
    * /var/lib/kea

  * Starts Kea DHCPv4:  
    * Binary: /sbin/kea-dhcp4  
    * Config file: /tmp/kea\_config.json  
      * This file must be provided by the carbide-dhcp-config ConfigMap mounted at /tmp.

**Ports:**

* udp – containerPort: 67  
* metrics – containerPort: 1089

**Security context:**

* capabilities.add: \[ "SYS\_PTRACE" \]  
  * Allows debugging (e.g., gdb/strace) inside the container.

#### Volume mounts

Inside the container:

1. **SPIFFE certs**  
   * name: spiffe  
   * mountPath: /var/run/secrets/spiffe.io  
   * readOnly: true  
2. Used by env vars:  
   * FORGE\_ROOT\_CAFILE\_PATH \= /var/run/secrets/spiffe.io/ca.crt  
   * FORGE\_CLIENT\_CERT\_PATH \= /var/run/secrets/spiffe.io/tls.crt  
   * FORGE\_CLIENT\_KEY\_PATH \= /var/run/secrets/spiffe.io/tls.key  
3. **Config**  
   * name: config  
   * mountPath: /tmp  
4. The carbide-dhcp-config ConfigMap is mounted here. It must include the Kea config file as /tmp/kea\_config.json.

#### Volumes

1. spiffe  
   * Type: Secret  
   * secret.secretName: carbide-dhcp-certificate  
   * Contents: TLS keypair \+ CA roots created by cert-manager from the Certificate resource.

2. config  
   * Type: ConfigMap  
   * configMap.name: carbide-dhcp-config  
   * defaultMode: 420 (octal 0644\)

#### Environment variables

These are all defined inline in deployment.yaml

1. FORGE\_ROOT\_CAFILE\_PATH  
   * **Value:** /var/run/secrets/spiffe.io/ca.crt  
   * **Purpose:** Path to CA certificate used to validate Forge/peer TLS endpoints.

2. FORGE\_CLIENT\_CERT\_PATH  
   * **Value:** /var/run/secrets/spiffe.io/tls.crt  
   * **Purpose:** Client certificate used by DHCP component to authenticate as carbide-dhcp to Forge or other services.

3. FORGE\_CLIENT\_KEY\_PATH  
   * **Value:** /var/run/secrets/spiffe.io/tls.key  
   * **Purpose:** Private key corresponding to FORGE\_CLIENT\_CERT\_PATH.

4. RUST\_BACKTRACE  
   * **Value:** "full"  
   * **Purpose:** Produce full stack traces when Rust code panics inside the image.

5. RUST\_LIB\_BACKTRACE  
   * **Value:** "0"  
   * **Purpose:** Controls backtrace behavior for Rust libraries (per Rust standard library docs).

**How to customize env vars:**

* For different cert paths or mounting locations:  
  * Adjust the volume mount path and update these env vars accordingly.  
* For debugging:  
  * Set RUST\_BACKTRACE to 1 or full (already full).  
  * Adjust RUST\_LIB\_BACKTRACE to enable library backtraces if needed.

### **7.8.2 Carbide DNS**

The Kubernetes manifests for the carbide-dns service and all its configurations are located in the following path in the carbide directory: deploy/carbide-base/dns. It corresponds to the following directory structure:

| deploy/carbide-base/dns ├── certificate.yaml ├── kustomization.yaml ├── role.yaml ├── rolebinding.yaml ├── service-account.yaml ├── service.yaml └── statefulset.yaml |
| :---- |

The carbide-dns service handles DNS (domain name service) queries from the site-controller services and managed hosts and is authoritative for them. It works in tandem with the unbound off-the-shelf DNS service for hosts outside of a carbide installation.

#### Primary Resources

- **Workloads:**  
  - StatefulSet/carbide-dns  
- **Services:**  
  - Service/carbide-dns  
- **TLS & Identity:**  
  - Certificate/carbide-dns-certificate  
- **RBAC:**  
  - ServiceAccount/carbide-dns  
  - Role/carbide-dns  
  - RoleBinding/carbide-dns

#### External Dependencies You Must Provide

- **Configuration:**  
  - ConfigMap/carbide-dns

#### Kustomization Entrypoint

The kustomization.yaml is the entrypoint that wires all carbide-dns manifests together and applies common labels so everything is consistently addressable.

The following **labels** are applied:

- app.kubernetes.io/name: carbide-dns  
- app.kubernetes.io/component: dns

Both includeSelectors: true and includeTemplates: true are set, so the above labels are applied to the Pods created by StatefulSets or Deployments as well as Service- and other selectors.

The following **resources** are included in the kustomization:

- certificate.yaml  
- statefulset.yaml  
- role.yaml  
- rolebinding.yaml  
- service-account.yaml  
- service.yaml

#### StatefulSet

##### Overview

| Name | carbide-dns |
| :---- | :---- |
| **Image** | nvcr.io/nvidian/nvforge/nvmetal-carbide:latest Can be overwritten with a pinned version in an overlay |
| **Ports** | udp/53, tcp/53 Default DNS ports |
| **Security Context** | SYS\_PTRACE Allows debugging (e.g. gdp/strace) inside of the container |

##### Volume Mounts

| Name | Description |
| :---- | :---- |
| spiffe | Certificate data used by carbide-dns to authenticate with carbide-api. Mounted at /var/run/secrets/spiffe.io and used by the environment variable FORGE\_ROOT\_CA\_PATH. |

##### Volumes

| Name | Description |
| :---- | :---- |
| spiffe | Mounts the secret carbide-dns-certificate which is created by Cert Manager as a result of the Certificate carbide-dns-certificate and contains a TLS keypair and the CA root.  |

##### Environment variables

| Name | Description |
| :---- | :---- |
| CARBIDE\_API | Extracted from the ConfigMap carbide-dns and contains the service URL for the carbide-api. Typically this is https://carbide-api.\<namespace\>.svc.cluster.local:1079. |
| FORGE\_ROOT\_CA\_PATH | Path to the CA certificate used to validate carbide/pee TLS endpoints. The value works in tandem with the spiffe volume mount and is therefore /var/run/secrets/spiffe.io/ca.crt. |
| RUST\_BACKTRACE | The value “full” causes full stack traces when Rust code panics inside the container. To disable, set the value to “0”. |
| RUEST\_LIB\_BACKTRACE | Controls the backtrace behavior for Rust libraries (see [https://doc.rust-lang.org/std/backtrace/index.html\#environment-variables](https://doc.rust-lang.org/std/backtrace/index.html#environment-variables)).  |

### **7.8.3 Carbide Hardware Health**

The Kubernetes manifests for the carbide-hardware-health service and all its configurations are located in the following path in the carbide directory: deploy/carbide-base/hardware-health. It corresponds to the following directory structure:

| deploy/carbide-base/hardware-health ├── certificate.yaml ├── deployment.yaml ├── kustomization.yaml ├── role.yaml ├── rolebinding.yaml ├── service-account.yaml └── service.yaml |
| :---- |

The carbide-hardware-health service scrapes all host and DPU BMCs known by carbide for system health information. It extracts measurements like fan speeds, temperatures and leak indicators. These measurements are emitted as prometheus metrics on a metrics endpoint on port 9009 for monitoring. In addition, the service calls the carbide-core API to set health alerts based on issues identified with the metrics.

#### Primary Resources

- **Workloads:**  
  - Deployment/carbide-hardware-health  
- **Services:**  
  - Service/carbide-hardware-health  
- **TLS & Identity:**  
  - Certificate/carbide-hardware-health-certificate  
- **RBAC:**  
  - ServiceAccount/carbide-hardware-health  
  - Role/carbide-hardware-health  
  - RoleBinding/carbide-hardware-health

#### Kustomization Entrypoint

The kustomization.yaml is the entrypoint that wires all carbide-hardware-health manifests together and applies common labels so everything is consistently addressable.

The following **labels** are applied:

- app.kubernetes.io/name: carbide-hardware-health  
- app.kubernetes.io/component: hardware-health

Both includeSelectors: true and includeTemplates: true are set, so the above labels are applied to the Pods created by StatefulSets or Deployments as well as Service- and other selectors.

The following **resources** are included in the kustomization:

- certificate.yaml  
- deployment.yaml  
- role.yaml  
- rolebinding.yaml  
- service-account.yaml  
- service.yaml

#### Deployment

##### Overview

| Name | carbide-hardware-health |
| :---- | :---- |
| **Image** | nvcr.io/nvidian/nvforge/nvmetal-carbide:latest Can be overwritten with a pinned version in an overlay |
| **Ports** | tcp/9009 Metrics port that exposes Prometheus metrics at /metrics. |
| **Security Context** | SYS\_PTRACE Allows debugging (e.g. gdp/strace) inside of the container |

##### Volume Mounts

| Name | Description |
| :---- | :---- |
| spiffe | Certificate data used by carbide-hardware-health to authenticate with carbide-api. Mounted at /var/run/secrets/spiffe.io. |

##### Volumes

| Name | Description |
| :---- | :---- |
| spiffe | Mounts the secret carbide-hardware-health-certificate which is created by Cert Manager as a result of the Certificate carbide-hardware-health-certificate and contains a TLS keypair and the CA root.  |

##### Environment variables

| Name | Description |
| :---- | :---- |
| RUST\_BACKTRACE | The value “full” causes full stack traces when Rust code panics inside the container. To disable, set the value to “0”. |
| RUEST\_LIB\_BACKTRACE | Controls the backtrace behavior for Rust libraries (see [https://doc.rust-lang.org/std/backtrace/index.html\#environment-variables](https://doc.rust-lang.org/std/backtrace/index.html#environment-variables)).  |

### **7.8.4 Nginx**

The Kubernetes manifests for the nginx service and all its configurations are located in the following path in the carbide directory: deploy/carbide-base/ngnix. It corresponds to the following directory structure:

| deploy/carbide-base/nginx ├── certificate.yaml ├── deployment.yaml ├── files │   └── nginx.conf ├── kustomization.yaml ├── service-account.yaml └── service.yaml |
| :---- |

The ngnix service is responsible for serving boot artifacts to managed hosts and DPUs when they network boot using iPXE. It hosts the needed boot images and metadata and works in tandem with the carbide-pxe service.

#### Primary Resources

- **Workloads:**  
  - Deployment/nginx  
- **Services:**  
  - Service/ngnix  
- **Configuration:**  
  - ConfigMap/ngnix-config  
- **TLS & Identity:**  
  - Certificate/nginx-certificate  
- **RBAC:**  
  - ServiceAccount/ngnix

#### Kustomization Entrypoint

The kustomization.yaml is the entrypoint that wires all nginx manifests together and applies common labels so everything is consistently addressable.

The following **labels** are applied:

- app.kubernetes.io/name: nginx  
- app.kubernetes.io/component: nginx

Both includeSelectors: true and includeTemplates: true are set, so the above labels are applied to the Pods created by StatefulSets or Deployments as well as Service- and other selectors.

The following **resources** are included in the kustomization:

- certificate.yaml  
- deployment.yaml  
- service-account.yaml  
- service.yaml

In addition, a generator creates the ConfigMap nginx-config from the static file located at files/nginx.conf that configures the ngnix web server.

#### Deployment

##### Overview

| Name | nginx |
| :---- | :---- |
| **Image** | nvcr.io/nvidian/nvforge/nvmetal-carbide:latest Can be overwritten with a pinned version in an overlay |
| **Ports** | tcp/80 Default HTTP port |
| **Security Context** | SYS\_PTRACE Allows debugging (e.g. gdp/strace) inside of the container |

##### Volume Mounts

| Name | Description |
| :---- | :---- |
| spiffe | Certificate data not used by anything. Mounted at /var/run/secrets/spiffe.io. |
| nginx-config | Mounts the nginx configuration from the ConfigMap ngnix-config at /etc/nginx. |

##### Volumes

| Name | Description |
| :---- | :---- |
| spiffe | Mounts the secret nginx-certificate which is created by Cert Manager as a result of the Certificate nginx-certificate and contains a TLS keypair and the CA root. |
| nginx-config | Mounts the ConfigMap nginx-config used by the nginx program as its main configuration file. |

### **7.8.5 Carbide NTP**

The Kubernetes manifests for the carbide-ntp service and all its configurations are located in the following path in the carbide directory: deploy/carbide-base/ntp. It corresponds to the following directory structure:

| deploy/carbide-base/ntp ├── kustomization.yaml ├── service.yaml └── statefulset.yaml |
| :---- |

The carbide-ntp service handles NTP (network time protocol) queries from the site-controller services and managed hosts.

#### Primary Resources

- **Workloads:**  
  - StatefulSet/carbide-ntp  
- **Services:**  
  - Service/carbide-ntp

#### Kustomization Entrypoint

The kustomization.yaml is the entrypoint that wires all carbide-ntp manifests together and applies common labels so everything is consistently addressable.

The following **labels** are applied:

- app.kubernetes.io/name: carbide-ntp  
- app.kubernetes.io/component: ntp

Both includeSelectors: true and includeTemplates: true are set, so the above labels are applied to the Pods created by StatefulSets or Deployments as well as Service- and other selectors.

The following **resources** are included in the kustomization:

- statefulset.yaml  
- service.yaml

#### StatefulSet

##### Overview

| Name | carbide-ntp |
| :---- | :---- |
| **Image** | dockurr/chrony:latest Can be overwritten with a pinned version in an overlay. |
| **Ports** | udp/123 Default NTP port |
| **PodAntiAffinity** | Ensures that the carbide-ntp replicas are spread across Kubernetes nodes and don’t run on the same node.  |

##### Environment variables

| Name | Description |
| :---- | :---- |
| NTP\_SERVERS | Defines the list of NTP servers to use. You can specify local NTP servers or use public NTP servers as long as they are accessible from your network. |
| NTP\_DIRECTIVES | Enables response rate limiting for NTP packets. See [https://chrony-project.org/doc/latest/chrony.conf.html](https://chrony-project.org/doc/latest/chrony.conf.html) for available directives. |

### **7.8.6 Carbide PXE**

The Kubernetes manifests for the carbide-pxe service and all its configurations are located in the following path in the carbide directory: deploy/carbide-base/pxe. It corresponds to the following directory structure:

| deploy/carbide-base/pxe ├── certificate.yaml ├── deployment.yaml ├── kustomization.yaml ├── metrics-service.yaml ├── role.yaml ├── rolebinding.yaml ├── service-account.yaml └── service.yaml |
| :---- |

The carbide-pxe (iPXE) service provides boot artifacts like iPXE scripts, iPXE user-data and OS images to managed hosts at boot time over HTTP. It determines which OS data to provide for a specific host by requesting the respective data from the carbide core. It works in tandem with nginx to serve the boot images over HTTP.

#### Primary Resources

- **Workloads:**  
  - Deployment/carbide-pxe  
- **Services:**  
  - Service/carbide-pxe  
  - Service/carbide-pxe-metrics  
- **TLS & Identity:**  
  - Certificate/carbide-pxe-certificate  
- **RBAC:**  
  - ServiceAccount/carbide-pxe  
  - Role/carbide-pxe  
  - RoleBinding/carbide-pxe

#### External Dependencies You Must Provide

- **Configuration**:  
  - ConfigMap/carbide-pxe-env-config

#### Kustomization Entrypoint

The kustomization.yaml is the entrypoint that wires all carbide-pxe manifests together and applies common labels so everything is consistently addressable.

The following **labels** are applied:

- app.kubernetes.io/name: carbide-pxe  
- app.kubernetes.io/component: pxe

Both includeSelectors: true and includeTemplates: true are set, so the above labels are applied to the Pods created by StatefulSets or Deployments as well as Service- and other selectors.

The following **resources** are included in the kustomization:

- certificate.yaml  
- deployment.yaml  
- role.yaml  
- rolebinding.yaml  
- service-account.yaml  
- service.yaml  
- metrics-service.yaml

#### Deployment

##### Overview

| Name | carbide-pxe |
| :---- | :---- |
| **Image** | nvcr.io/nvidian/nvforge/nvmetal-carbide:latest Can be overwritten with a pinned version in an overlay |
| **Ports** | tcp/8080 HTTP port |
| **Security Context** | SYS\_PTRACE Allows debugging (e.g. gdp/strace) inside of the container |

##### Volume Mounts

| Name | Description |
| :---- | :---- |
| spiffe | Certificate data used by carbide-pxe to authenticate with carbide-api. Mounted at /var/run/secrets/spiffe.io and used by the environment variables FORGE\_ROOT\_CAFILE\_PATH, FORGE\_CLIENT\_CERT\_PATH, FORGE\_CLIENT\_KEY\_PATH. |
| config | Configuration data for carbide-pxe mounted at /tmp/carbide.  |

##### Volumes

| Name | Description |
| :---- | :---- |
| spiffe | Mounts the secret carbide-dns-certificate which is created by Cert Manager as a result of the Certificate carbide-dns-certificate and contains a TLS keypair and the CA root.  |
| config | Mounts the ConfigMap `carbide-pxe-config`, specifically the `Rocket.toml` file. |

##### Environment variables

| Name | Description |
| :---- | :---- |
| ROCKET\_CONFIG | Path to the rocket configuration file. Use the same path as the mountpoint of the `config` mount point. |
| ROCKET\_TEMPLATE\_DIR | Path to where rocket templates are stored. |
| ROCKET\_ENV | Type of environment to use for rocket. Use “production” for a production environment. |
| FORGE\_ROOT\_CAFILE\_PATH | Path to the CA certificate used to validate carbide/peer TLS endpoints. The value works in tandem with the spiffe volume mount and is therefore /var/run/secrets/spiffe.io/ca.crt. |
| FORGE\_CLIENT\_CERT\_PATH | Path to the CA certificate used to validate carbide/peer TLS endpoints. The value works in tandem with the spiffe volume mount and is therefore /var/run/secrets/spiffe.io/tls.crt. |
| FORGE\_CLIENT\_KEY\_PATH | Path to the CA certificate used to validate carbide/peer TLS endpoints. The value works in tandem with the spiffe volume mount and is therefore /var/run/secrets/spiffe.io/tls.key. |
| RUST\_BACKTRACE | The value “full” causes full stack traces when Rust code panics inside the container. To disable, set the value to “0”. |
| RUEST\_LIB\_BACKTRACE | Controls the backtrace behavior for Rust libraries (see [https://doc.rust-lang.org/std/backtrace/index.html\#environment-variables](https://doc.rust-lang.org/std/backtrace/index.html#environment-variables)).  |
| … | Environment variables that are defined in the ConfigMap `carbide-pxe-env-config` that is defined externally. |

### **7.8.7 ssh Console RS**

carbide-ssh-console-rs exposes an SSH endpoint that proxies to machine consoles (DPUs), authenticates using SSH certs, and talks to carbide-api over mTLS using SPIFFE identities. It also logs console output and ships those logs to Loki via an OpenTelemetry Collector sidecar.

| .├── certificate.yaml├── config-files│   ├── config.toml│   └── otelcol-config.yaml├── deployment.yaml├── kustomization.yaml├── metrics-service.yaml├── role.yaml├── rolebinding.yaml├── secrets│   └── ssh\_host\_key.enc.yaml├── service-account.yaml├── service.yaml└── ssh-host-key-secret-generator.yaml |
| :---- |

* **Workload**  
  * Deployment/carbide-ssh-console-rs (two containers: ssh-console-rs \+ loki-log-collector)  
* **Networking**  
  * Service/carbide-ssh-console-rs (SSH on TCP/22)  
  * Service/carbide-ssh-console-rs-metrics (metrics on TCP/9009)  
* **Config**  
  * ConfigMap/ssh-console-rs-config-files → /etc/forge/ssh-console/config.toml  
  * ConfigMap/ssh-console-rs-otelcol-config → /etc/otelcol-contrib/config.yaml  
* **Identity & TLS**  
  * Certificate/carbide-ssh-console-rs-certificate → Secret/carbide-ssh-console-rs-certificate  
  * SPIFFE URI: spiffe://forge.local/default/sa/carbide-ssh-console-rs  
* **SSH host key**  
  * Secret/ssh-host-key (generated from secrets/ssh\_host\_key.enc.yaml via ksops)  
* **RBAC**  
  * ServiceAccount/carbide-ssh-console-rs  
  * Role/carbide-ssh-console-rs  
  * RoleBinding/carbide-ssh-console-rs  
* **Kustomize/ksops**  
  * kustomization.yaml (wires all resources \+ configMapGenerators)  
  * ssh-host-key-secret-generator.yaml (ksops hook to decrypt the host key secret)

**External dependencies:**

* ClusterIssuer/vault-forge-issuer (cert-manager)  
* Loki reachable at http://loki.loki.svc.cluster.local:3100/loki/api/v1/push  
* carbide-api available at https://carbide-api.forge-system.svc.cluster.local:1079  
* Age/SOPS key for decrypting ssh\_host\_key.enc.yaml

#### Kustomization Entrypoint (kustomization.yaml)

### **Resources**

Kustomize includes:

* service.yaml  
* certificate.yaml  
* deployment.yaml  
* metrics-service.yaml  
* role.yaml  
* rolebinding.yaml  
* Service-account.yaml

##### ConfigMap generators

| configMapGenerator:  \- name: ssh-console-rs-config-files    files:      \- config-files/config.toml  \- name: ssh-console-rs-otelcol-config    files:      \- config.yaml=config-files/otelcol-config.yaml |
| :---- |

This produces:

1. ConfigMap/ssh-console-rs-config-files  
   * Contains config.toml  
   * Mounted at /etc/forge/ssh-console/  
2. ConfigMap/ssh-console-rs-otelcol-config  
   * Contains config.yaml  
   * Mounted at /etc/otelcol-contrib/config.yaml

##### Secret generator (ksops)

| generators:  \- ./ssh-host-key-secret-generator.yaml |
| :---- |

This tells kustomize (via ksops) to:

* Decrypt secrets/ssh\_host\_key.enc.yaml using SOPS/age  
* Emit a Secret/ssh-host-key resource into the rendered output

#### Configuration Files (config-files/)

**config-files/config.toml**

This is the runtime config for the ssh-console-rs binary.

**Key knobs:**

* **Networking**  
  * **listen\_address** : "\[::\]:22"  
     SSH listen address (IPv6 all interfaces on port 22). Change if you want a different port or bind address.  
  * metrics\_address : "\[::\]:9009"  
     HTTP metrics address (port 9009). Used by metrics-service.

* **carbide-api connectivity**  
  * **carbide\_url** : "https://carbide-api.forge-system.svc.cluster.local:1079"  
     HTTPS endpoint of carbide-api. Must be reachable from this pod.

  * forge\_root\_ca\_path : "/var/run/secrets/spiffe.io/ca.crt"  
    Path to root CA cert for carbide-api

  * client\_cert\_path:  "/var/run/secrets/spiffe.io/tls.crt"   
    Client cert path to communicate with carbide-api  
*   
  * client\_key\_path: "/var/run/secrets/spiffe.io/tls.key"  
    Client certificate and key for mTLS, also from the SPIFFE secret.

* **SSH host key**  
  * **host\_key**: "/etc/ssh/ssh\_host\_ed25519\_key"  
     Path to the SSH host private key (/etc/ssh/ssh\_host\_ed25519\_key). This comes from Secret/ssh-host-key.

* **Access control**  
  * **dpus**: true  
     Enables SSH access to DPU consoles.

  * insecure   
     **Must be false in production.** If set to true, all incoming SSH clients are allowed (for dev/tests).

  * openssh\_certificate\_ca\_fingerprints   
    Allowed CA fingerprints for OpenSSH certs.  
    Signing CA fingerprints for openssh certificates. Defaults to one that's valid for production (nvinit certs)

  * admin\_certificate\_role   
     Role in the SSH certificate Key ID that grants admin-level console access.  
    Roles which determine admin access (logins with an openssh certificate, signed by the above fingerprints, containing this role in its Key ID, are considered admins and can log into machines directly.) (Currently swngc-forge-admins)

* **IPMI / legacy**  
  * **insecure\_ipmi\_ciphers**   
     Allows insecure IPMI crypto (useful for simulators; should be false in prod).

* **Authorized keys (dev)**  
  * **authorized\_keys\_path** (No used in production)  
    Optional path to an authorized\_keys file for dev. In production, prefer OpenSSH certificates.

* **Polling and logging**

  * **api\_poll\_interval**   
     How often to poll carbide-api for machines.

  * console\_logging\_enabled   
     Whether to write per-machine logs.

  * console\_logs\_path   
     Filesystem path for console logs. This is backed by the console-logs emptyDir volume.

**config-files/otelcol-config.yaml**

This configures the OpenTelemetry Collector sidecar (loki-log-collector) for log shipping

Key sections:

* **extensions.file\_storage**  
  * directory: /var/lib/otelcol/filelog-checkpoints  
     Stores checkpoints so filelog receiver knows where it left off after restarts.  
* **receivers.filelog**  
  * include:  
    * /var/log/consoles/\*.log  
    * /var/log/consoles/\*.log.\*  
  * start\_at: beginning   
    read logs from the start.  
  * storage: file\_storage   
    uses the extension above.  
  * operators:  
    * regex\_parser with regex:  
       ^(?P\<machineid\>\[a-z0-9\]+)\_(?P\<bmc\_ip\>\[^\_\]+).log$  
       Parses the filename into:  
      * machineid  
      * bmc\_ip

* **processors**  
  * attributes:  
    * Adds/updates attributes:  
      * exporter \= "forge-ssh-console-rs"  
      * loki.attribute.labels \= "machineid,exporter"  
      * loki.format \= "raw"  
  * batch – batching for efficiency.  
  * memory\_limiter:  
    * limit\_mib: 4096  
    * spike\_limit\_mib: 1024  
* **exporters.loki**  
  * endpoint: "http://loki.loki.svc.cluster.local:3100/loki/api/v1/push"  
  * headers:  
    * "X-Scope-OrgID": forge (multi-tenant Loki label)  
* **service**  
  * extensions: \[file\_storage\]  
  * pipelines.logs:  
    * receivers: \[filelog\]  
    * processors: \[attributes, batch\]  
    * exporters: \[loki\]

**Configurable knobs:**

* Loki endpoint and X-Scope-OrgID  
* Memory limits for Otelcol  
* File patterns and regex for log filenames  
* Additional labels via attributes processor

#### Deployment (deployment.yaml)

Metadata & basic spec

* **Kind:** Deployment  
* **Name:** carbide-ssh-console-rs  
* **Namespace:** default  
* **Spec:**  
  * replicas: 1  
  * strategy.type: Recreate  
     (Only one pod at a time; avoids double-binding port 22 / state issues.)  
  * selector.matchLabels.app.kubernetes.io/name: carbide-ssh-console-rs

### **4.2 Pod spec**

* Uses ServiceAccount/carbide-ssh-console-rs for SPIFFE identity and RBAC.  
* Uses imagepullsecret for pulling the nvmetal-carbide image.  
  Get’s distributed by the ESO

#### 4.3 Containers

**ssh-console-rs (main container)**

* **Image:** nvcr.io/nvidian/nvforge/nvmetal-carbide:latest  
* **Command:** \["/opt/carbide/ssh-console"\]  
* **Args:** \["run", "-c", "/etc/forge/ssh-console/config.toml"\]  
  * Reads config from the mounted config.toml.

* **Ports:**  
  * ssh: containerPort: 22, protocol: TCP

* **Readiness probe:**  
  * tcpSocket.port: 22  
  * initialDelaySeconds: 1  
  * periodSeconds: 30  
  * failureThreshold: 3

* **Security context:**  
  * Adds capability IPC\_LOCK to let the SSH engine use mlock(2) (keep sensitive data out of swap).

* **Volume mounts:**  
  * ssh-console-rs-config-files → /etc/forge/ssh-console  
  * spiffe → /var/run/secrets/spiffe.io (read-only)  
  * ssh-host-key → /etc/ssh/ssh\_host\_ed25519\_key (via subPath: ssh\_host\_ed25519\_key, read-only)  
  * ssh-host-key-pub → /etc/ssh/ssh\_host\_ed25519\_key.pub (via subPath: ssh\_host\_ed25519\_key\_pub, read-only)  
  * console-logs → /var/log/consoles

**loki-log-collector (otelcol sidecar)**

* **Image:**  
  ghcr.io/open-telemetry/opentelemetry-collector-releases/opentelemetry-collector-contrib:0.81.0  
* **Args:** \["--config=/etc/otelcol-contrib/config.yaml"\]  
* **Volume mounts:**  
  * otelcol-config → /etc/otelcol-contrib (read-only)  
  * console-logs → /var/log/consoles  
  * otelcol-file-log-checkpoints → /var/lib/otelcol/filelog-checkpoints

**Volumes**

* **ssh-console-rs-config-files**  
  **configMap:**  
  * **name: ssh-console-rs-config-files**  
* **Spiffe**  
  **Secret:**  
  * **secretName: carbide-ssh-console-rs-certificate**  
* **Ssh-host-key**  
  **Secret:**  
  * **secretName: ssh-host-key**  
  * **defaultMode: 0400**  
* **Ssh-host-key-pub**  
  **Secret:**  
  * **secretName: ssh-host-key**  
  * **defaultMode: 0444**  
* **Console-logs**  
  **emptyDir: {}**

* **Otelcol-file-log-checkpoints**  
  **emptyDir: {}**

* **Otelcol-config**  
  **configMap:**  
  * **name: ssh-console-rs-otelcol-config**

#### Services

**SSH service (service.yaml)**

* **Exposes SSH on port 22\.**

| apiVersion: v1kind: Servicemetadata:  name: carbide-ssh-console-rsspec:  ports:    \- port: 22      name: ssh      protocol: TCP |
| :---- |

**Metrics service (metrics-service.yaml)**

* **Exposes metrics on port 9009 (matching metrics\_address in config).**

| apiVersion: v1kind: Servicemetadata:  name: carbide-ssh-console-rs-metrics  namespace: default  labels:    app: carbide-ssh-console-rs    app.kubernetes.io/metrics: carbide-ssh-console-rsspec:  ports:    \- port: 9009      name: http      protocol: TCP      targetPort: 9009 |
| :---- |

#### TLS Certificate (certificate.yaml)

**Behavior:**

* cert-manager uses ClusterIssuer/vault-forge-issuer to issue a cert.  
* The resulting Kubernetes secret:  
  * Name: carbide-ssh-console-rs-certificate  
  * Namespace: forge-system

**Usage:**

* Mounted into the pod as volume spiffe at /var/run/secrets/spiffe.io.  
* Paths in config.toml reference:  
  * /var/run/secrets/spiffe.io/ca.crt  
  * /var/run/secrets/spiffe.io/tls.crt  
  * /var/run/secrets/spiffe.io/tls.key

**Configurable parts:**

* dnsNames: adjust if you change service DNS.  
* uris: SPIFFE URI; adjust trust domain or namespace if needed.  
* issuerRef: change if you use a different ClusterIssuer.  
* Key algorithm/size if you require different crypto.

#### SSH Host Key Secret & ksops (ssh\_host\_key.enc.yaml)

| ├── secrets│   └── ssh\_host\_key.enc.yaml |
| :---- |

Before encryption, the SSH host key secret logically looks like:

| apiVersion: v1kind: Secretmetadata:  name: ssh-host-key  namespace: forge-system  \# (must match your desired namespace)type: OpaquestringData:  ssh\_host\_ed25519\_key: |    \-----BEGIN OPENSSH PRIVATE KEY-----    PRIVATE KEY HERE    \-----END OPENSSH PRIVATE KEY-----  ssh\_host\_ed25519\_key\_pub: |    ssh-ed25519 PUBLIC KEY HERE |
| :---- |

**Generating the SSH host keys**

| mkdir \-p host\_keys/etc/sshssh-keygen \-A \-f ./host\_keys |
| :---- |

This produces:

| host\_keys/└── etc    └── ssh        ├── ssh\_host\_ecdsa\_key        ├── ssh\_host\_ecdsa\_key.pub        ├── ssh\_host\_ed25519\_key        ├── ssh\_host\_ed25519\_key.pub        ├── ssh\_host\_rsa\_key        └── ssh\_host\_rsa\_key.pub |
| :---- |

You then:

1. Copy the **contents** of ssh\_host\_ed25519\_key into ssh\_host\_ed25519\_key in stringData (including the \-----BEGIN/END lines).

2. Copy the line from ssh\_host\_ed25519\_key.pub into ssh\_host\_ed25519\_key\_pub.

After filling the plaintext secret, you:

* Run sops with your age key to encrypt secrets/ssh\_host\_key.enc.yaml.

* ksops uses that file plus your age private key (in \~/.config/sops/age/...) to decrypt at build time and emit a Secret/ssh-host-key resource.

### **7.8.8 UFM \-\> Milad**

Sd

## **7.9 Carbide Unbound Base(recursive DNS)**

#### Role

carbide-unbound runs an Unbound recursive DNS resolver inside the cluster. It:

* Answers DNS for Carbide and other internal workloads  
* Forwards queries to your chosen upstream resolvers  
* Exposes basic Prometheus metrics (via the unbound\_exporter sidecar)

In our reference environment this runs alongside the rest of Carbide core (e.g., in forge-system), but the manifests don’t hard‑code a namespace — it will deploy into whatever namespace you apply it to.

---

#### Kustomize layout

From the carbide-unbound-base directory:

| carbide-unbound-base/  local.conf.d/    access\_control.conf    extended\_statistics.conf    forge\_local\_unknowndomain.conf    patchme.conf    verbosity.conf  deployment.yaml  service.yaml  kustomization.yaml  unbound.env |
| :---- |

kustomization.yaml generates two ConfigMaps and wires them into the Deployment:

* unbound-envvars from unbound.env  
* unbound-local-config from the files in local.conf.d/

---

### **Behavior in our reference environment**

#### Deployment

* Name: carbide-unbound  
* Containers:  
  * unbound  
    * Image (reference only): nvcr.io/nvidian/nvforge/unbound:3b089e6  
    * Ports: UDP/53 (dns-udp), TCP/53 (dns-tcp)  
    * Liveness / readiness: TCP probe on port 53  
    * Volume mounts:  
      * /etc/unbound/local.conf.d → ConfigMap unbound-local-config  
      * /etc/unbound/keys → emptyDir (for control keys)  
    * EnvFrom: ConfigMap unbound-envvars  
  * unbound-metrics  
    * Image (reference only): nvcr.io/nvidian/nvforge/unbound\_exporter:7561033  
    * Env: KEYS\_DIR=/etc/unbound/keys  
    * Volume mount: /etc/unbound/keys (read‑only)  
    * Port: 9167 (Prometheus metrics)

#### Service (internal only)

* Name: carbide-unbound (via service.yaml)  
* Type: ClusterIP  
* Ports:  
  * UDP/53 and TCP/53 for DNS  
  * (Metrics scraped via pod annotations on port 9167\)

---

#### **Config knobs SAs / customers must review**

Most of the customization is in local.conf.d/\* and unbound.env.

#### **1\. DNS access control**

**File:** local.conf.d/access\_control.conf

**Content (reference):**

| server:    access-control: 0.0.0.0/0 allow |
| :---- |

* This effectively allows DNS queries from anywhere that can reach the Service.  
* **Customer action:** tighten this to the cluster / node CIDRs or site networks you actually want to serve.  
   Examples:

| server:    access-control: 10.0.0.0/8 allow    access-control: 192.168.0.0/16 allow    access-control: 0.0.0.0/0 refuse |
| :---- |

#### **2\. Extended statistics & verbosity**

**Files:**

* local.conf.d/extended\_statistics.conf

| server:    extended-statistics: yes |
| :---- |

* local.conf.d/verbosity.conf (not shown above, but included in the ConfigMap)

These control Unbound’s statistics and log verbosity.

* **Customer action:**  
  * Leave extended-statistics: yes if you want richer metrics, or set to no to trim.  
  * Adjust verbosity (verbosity: 1/2/3…) in verbosity.conf per your logging policy.

##### **3\. “unknowndomain” handling (don’t leak bogus queries)**

**File:** local.conf.d/forge\_local\_unknowndomain.conf

| local\-zone: "unknowndomain." staticlocal\-data: "unknowndomain. 10800 IN NS localhost."local\-data: "unknowndomain. 10800 IN SOA localhost. nobody.invalid. 1 3600 1200 604800 10800" |
| :---- |

* This ensures queries for the hard‑coded unknowndomain. (used when a VPC has no domain) are handled locally rather than forwarded.

* **Customer action:** generally none; keep this as‑is unless you intentionally change Carbide’s default domain behavior.

#### **4\. Upstream forwarders (critical)**

**File:** local.conf.d/patchme.conf

| \# This file's contents are intended to be broken if used at runtime. The base\# configMapGenerator should be merged with a later one that replaces this file.include: patchme/patchme.conf |
| :---- |

* In the base, patchme.conf is intentionally “broken” — it just includes another non‑existent file.  
* **Customer action:** replace this with real Unbound configuration for your upstream DNS:

Example:

| server:    do\-ip4: yes    do\-ip6: no    do\-udp: yes    do\-tcp: yesforward-zone:    name: "."    forward-addr: 10.0.0.10       \# your internal resolver    forward-addr: 10.0.0.11       \# optional second resolver |
| :---- |

Or, if you’re layering via your own Kustomize overlay, override forwarders.conf in your overlay instead of editing the base.

#### **5\. Environment defaults**

**File:** unbound.env

| LOCAL\_CONFIG\_DIR=/etc/unbound/local.conf.dBROKEN\_DNSSEC=1UNBOUND\_CONTROL\_DIR=/etc/unbound/keys |
| :---- |

* LOCAL\_CONFIG\_DIR should match the volume mount path; normally no change needed.

* BROKEN\_DNSSEC=1 hints that DNSSEC validation may be relaxed.

  * Set BROKEN\_DNSSEC=0 if you run with correct DNSSEC upstream and want strict validation.

* UNBOUND\_CONTROL\_DIR is where control keys and sockets live; normally leave as default.

---

#### **Images & registry**

In our reference cluster we used:

* nvcr.io/nvidian/nvforge/unbound:3b089e6

* nvcr.io/nvidian/nvforge/unbound\_exporter:7561033

For customer deployments:

1. **Build or mirror** these images to the customer’s registry (or use their own Unbound / exporter images).

2. Update deployment.yaml (or a kustomization.yaml images: block) so that:

   * image: \<REGISTRY\>/\<PROJECT\>/unbound:\<TAG\>

   * image: \<REGISTRY\>/\<PROJECT\>/unbound\_exporter:\<TAG\>

3. Ensure imagePullSecrets refers to a secret that can authenticate to that registry, or remove it if their registry doesn’t require auth.

---

#### **How to apply carbide-unbound**

Once:

* You’ve set appropriate access controls and upstream forwarders in local.conf.d/\*, and

* You’ve updated the Unbound / exporter image references to the customer’s registry,

the architect deploys this component with:

| kubectl apply \-k \<path-to\>/carbide-unbound-base |
| :---- |

This will create:

* The unbound-envvars and unbound-local-config ConfigMaps  
* The carbide-unbound Deployment (Unbound \+ metrics sidecar)  
* The internal DNS Service exposing UDP/TCP 53 (and metrics scraped via annotations)

Carbide components can then be pointed at this Service as their recursive DNS (e.g., via Pod dnsConfig or node / cluster DNS settings, depending on how you integrate it.

**7.10 \-  Bases \- [Jasmeer Abdulvahid US](mailto:jabdulvahid@nvidia.com)**

## **7.11 FRR routing (frrouting-base)**

### **Role**

frrouting runs FRR BGP speakers inside the cluster. It’s used when Carbide needs to:

* Peer with ToR / upstream routers over BGP, and  
* Export / learn routes for management and/or bare‑metal networks.

This component is **optional** – if a customer already has an on‑prem routing solution that doesn’t require in‑cluster BGP, they can skip it.

---

### **Kustomize layout**

From the frrouting-base directory:

| frrouting-base/  files/    daemons          \# Which FRR daemons run    extra-init.sh    \# Entry helper: picks per‑pod config    vtysh.conf       \# vtysh CLI config (empty by default)  kustomization.yaml  service.yaml  statefulset.yaml |
| :---- |

kustomization.yaml creates:

* ConfigMap **frrouting-config** → mounted at /etc/frr\_pre (base FRR config files)  
* ConfigMap **frrouting-extra-init** → mounted at /usr/local/bin/frr-extra-init (the init script)

---

### **Behavior in our reference environment**

#### StatefulSet

* Name: frrouting  
* Replicas: 3  
* Pod labels: app.kubernetes.io/name: frrouting  
* Annotations: triggers reload when frrouting-config changes.  
* Pod anti‑affinity: spreads replicas across nodes (topologyKey: kubernetes.io/hostname).

#### Containers

1. **frr** (FRR routing daemon)  
   * Image (reference only): nvcr.io/nvidian/nvforge/frr:latest  
   * Command:  
     * /sbin/tini \-- /usr/local/bin/frr-extra-init/extra-init.sh /usr/lib/frr/docker-start  
   * Security:  
     * privileged: true (required for FRR to manage routing tables / sockets).  
   * Ports:  
     * TCP/179 (bgp)  
   * Volume mounts:  
     * /etc/frr\_pre → ConfigMap frrouting-config  
     * /usr/local/bin/frr-extra-init → ConfigMap frrouting-extra-init (script)  
     * /var/run/frr → emptyDir (sockets, runtime files)

2. **Init logic (extra-init.sh):**

   * Derives STATEFULSET\_INDEX from pod hostname.  
   * Copies everything from /etc/frr\_pre/ into /etc/frr/.  
   * If /etc/frr/frr.conf-\<INDEX\> exists, it copies that to /etc/frr/frr.conf.  
   * Optionally runs any additional scripts under ${EXTRA\_INIT\_PATH}.  
   * Then execs FRR’s docker-start entrypoint.  
3. This lets you ship per‑instance configs like frr.conf-0, frr.conf-1, etc., for different peers.  
4. **frr-exporter** (metrics)  
   * Image: ghcr.io/tynany/frr\_exporter  
   * Args include:  
     * \--frr.socket.dir-path=/frr\_sockets  
     * \--no-collector.ospf, \--no-collector.bfd, \--collector.bgp6  
   * Port: TCP/9342 (metrics)  
   * Mounts /frr\_sockets → same emptyDir as /var/run/frr so it can scrape FRR.

#### Service

* Name: frrouting  
* Type: **headless** (clusterIP: None)  
* Port: TCP/179 (bgp)

Peering routers will talk to each pod’s IP on TCP/179. How those IPs are exposed (pod IP vs. hostNetwork, etc.) is up to the customer’s overlay / environment.

---

### **Config knobs SAs / customers must adjust**

Most of the “real” routing configuration is intentionally **not** included in the base. The base only:

* Enables bgpd (BGP) in daemons  
* Provides a stub vtysh.conf  
* Wires the extra‑init mechanism

Customers must supply their own FRR config that matches their network design.

#### 1\. FRR daemons

**File:** files/daemons

| Key lines (reference)FRR\_NO\_ROOT="yes"MAX\_FDS=65536bgpd=yesbgpd\_options=" \--no\_kernel \--skip\_runas"\# everything else set to "no" |
| :---- |

* By default **only BGP** is enabled.  
* **Customer actions:**  
  * Keep bgpd=yes (BGP) if they need routing integration.  
  * Flip other daemons (ospfd, isisd, etc.) to yes only if they explicitly use them and understand the implications.

#### 2\. BGP configuration (critical)

**Config source:** files that ultimately end up under /etc/frr/:

* Base files in frrouting-config (from files/\*)  
* Optional per‑pod configs frr.conf-0, frr.conf-1, frr.conf-2, etc., if you add them via your own overlay.

The base ships **no BGP neighbors or networks** – vtysh.conf is empty and there is no frr.conf. Extra‑init will happily run with an empty config, but nothing will be advertised.

* **Customer actions:**  
  * Provide valid FRR config with:  
    * Local AS number(s)  
    * BGP neighbors (ToRs, upstream routers)  
    * Address families (IPv4 / IPv6)  
    * Prefixes to advertise and any route‑maps / communities  
  * Decide if each FRR instance should have identical config (same peers) or per‑instance configs:  
    * Common config for all pods: mount a frr.conf file into /etc/frr\_pre via your own ConfigMap.  
    * Per‑pod configs: create frr.conf-0, frr.conf-1, frr.conf-2 in the ConfigMap so extra‑init picks the right one based on the StatefulSet index.

Examples of where to layer this:

* Your own Kustomize overlay that extends frrouting-config with additional files.  
* A separate ConfigMap mounted at /etc/frr\_pre that overrides the base.

#### 3\. Replica count & placement

* spec.replicas: 3 in the StatefulSet.  
* Pod anti‑affinity ensures they land on different nodes.  
* **Customer actions:**  
  * Set replicas to the number of BGP speakers they actually want (often 2 or 3).  
  * Ensure there are at least as many nodes as replicas for anti‑affinity to succeed.

#### 4\. Routing integration & security

* The frr container runs **privileged**.  
* The manifest doesn’t use hostNetwork, so by default BGP runs on pod IPs.  
* **Customer actions:**  
  * Decide whether peering should be done to pod IPs or via a hostNetwork/DaemonSet‑style pattern (if they adapt the manifest).  
  * Ensure appropriate NetworkPolicies and firewall rules allow TCP/179 between FRR pods and their peers.  
  * Review cluster policies around privileged pods; tighten as required.

#### 5\. Images & registry

Reference images:

* nvcr.io/nvidian/nvforge/frr:latest (FRR)  
* ghcr.io/tynany/frr\_exporter (metrics)

For customer deployments:

* Mirror or build your own FRR image and exporter image in the customer’s registry.  
* Update the StatefulSet container image: fields (or add an images: block in a higher‑level kustomization.yaml) to use:  
  * \<REGISTRY\>/\<PROJECT\>/frr:\<TAG\>  
  * \<REGISTRY\>/\<PROJECT\>/frr-exporter:\<TAG\>  
* Keep or adjust imagePullSecrets depending on whether their registry requires authentication.

---

### **How to apply frrouting**

Once:

* The BGP configuration files are prepared (either shared frr.conf or per‑pod frr.conf-\<index\>), and  
* The FRR / exporter images are pointing to the customer’s registry,

the architect deploys FRR with:

| kubectl apply \-k \<path-to\>/frrouting-base |
| :---- |

This will create:

* The frrouting-config & frrouting-extra-init ConfigMaps  
* The frrouting headless Service on TCP/179  
* The frrouting StatefulSet (FRR \+ metrics exporter on each pod)

## **7.12 Forge System**

The forge-system directory defines the **Forge control-plane namespace** and the core wiring for services that need:

* A shared system namespace (forge-system)

* Standard config for:

  * carbide-api URL

  * Carbide database connection

  * Vault endpoint and mounts

* Cluster-internal identities (SPIFFE certificates) for key services

* **External access** via LoadBalancer services for DHCP, DNS, NTP, PXE, carbide-api, SSH console, FRR (BGP), and nginx

Directory hierarchy PATH/TO/CARBIDE-REPO/deploy/forge-system :

| forge-system├── external-services.yaml├── kustomization.yaml└── namespace.yaml |
| :---- |

###  **Kustomization**

* **namespace: forge-system**  
   All resources in this kustomization are defaulted into the forge-system namespace

* **generatorOptions.disableNameSuffixHash: true**  
   Generated ConfigMaps will not get hash suffixes (e.g., \-abcd1234). This is important because:  
  * Secrets/ConfigMaps are referenced by **fixed names** in deployments.

#### Included resources

This overlay pulls together:

* namespace.yaml → creates Namespace/forge-system  
* external-services.yaml → defines all the LoadBalancer services that expose various components externally  
* ../carbide-base → base manifests for carbide components (API, DHCP, DNS, PXE, etc.)  
  Documented earlier  
* ../frrouting-base → FRR routing stack (BGP)  
  Documented earlier  
* ../carbide-unbound-base → DNS infrastructure (Unbound / carbide-dns)  
  Documented earlier

Knob: These base paths are structural. To customize per-site, you typically

* Keep resources as-is  
* Patch specifics (certs, services, etc.) in overlays

#### Components (TODO)

#### ConfigMap Generators

* Name: carbide-dns  
* Behavior: create (won’t overwrite if already exists from another layer)  
* Key:  
  * CARBIDE\_API  
     Default: https://carbide-api.forge-system.svc.cluster.local:1079  
     Used by carbide-dns components to talk to the Carbide API.

Knobs:

* Change CARBIDE\_API if:  
  * You host carbide-api elsewhere

#### ConfigMap/forge-system-carbide-database-config

* Keys:  
  * DB\_HOST  
     Default: forge-pg-cluster.postgres.svc.cluster.local

  * DB\_PORT  
     Default: 5432

  * DB\_NAME  
     Default: forge\_system\_carbide

  * SECRET\_REF  
     Default: forge-system.carbide.forge-pg-cluster.credentials.postgresql.acid.zalan.do  
    (Reference to the Postgres credentials secret created by the Zalando operator)

These are consumed by Carbide components (like carbide-api, migration jobs, etc.) to build the actual Postgres connection string.  
Knobs for client:

* DB\_HOST / DB\_PORT  
  * If you run Postgres in a different namespace or cluster, point this to your service.

* DB\_NAME  
  * Change if you want a different database name on the same cluster.

* SECRET\_REF  
  * Update if your Postgres operator uses a different secret naming pattern.

#### ConfigMap/vault-cluster-info

* Keys:  
  * VAULT\_SERVICE  
     Default: https://vault.vault.svc.cluster.local:8200  
     (Vault endpoint for API calls)

  * FORGE\_VAULT\_MOUNT  
     Default: secrets – KV mount name

  * FORGE\_VAULT\_PKI\_MOUNT  
     Default: forgeca – PKI mount name for issuing certificates

These are read by Carbide services to talk to Vault (tokens usually supplied via secrets).  
Knobs:

* VAULT\_SERVICE – point to your Vault service DNS / port.  
* FORGE\_VAULT\_MOUNT – change if your KV mount uses a different path.  
* FORGE\_VAULT\_PKI\_MOUNT – change if PKI is mounted under a different path.

#### Patches Section

The patches: section customizes certificates and the Postgres cluster.  
Each patch uses a **strategic merge patch** targeted at a specific resource.

#### Certificate SPIFFE / DNS adjustments

For each Certificate resource, we override spec.dnsNames and spec.uris to match the forge-system namespace and trust domain (forge.local):

* carbide-dns-certificate  
* carbide-pxe-certificate  
* nginx-certificate  
* carbide-dhcp-certificate  
* carbide-api-certificate  
* carbide-hardware-health-certificate  
* Carbide-ssh-console-rs-certificate

Here is an example of carbide-dns-certificate

|  \- patch: |-     apiVersion: cert-manager.io/v1    kind: Certificate    metadata:      name: carbide-dns-certificate    spec:      dnsNames:        \- carbide-dns.forge-system.svc.cluster.local      uris:        \- spiffe://forge.local/forge-system/sa/carbide-dns   target:     group: cert-manager.io     version: v1     kind: Certificate     name: carbide-dns-certificate |
| :---- |

What this does:

* Ensures certificates issued for these services:  
  * Have the correct Kubernetes DNS name (namespace-qualified).  
  * Have the correct SPIFFE URI, including:  
    * Trust domain: forge.local  
    * Namespace: forge-system  
    * ServiceAccount name (e.g., carbide-api)

Knobs for clients:

* Trust domain  
  * Change spiffe://forge.local/... if your SPIFFE trust domain is something else (e.g. spiffe://mycorp.local/...).  
* Namespace  
  * If you deploy these components in a different namespace, update:  
    * dnsNames (e.g. carbide-api.\<namespace\>.svc.cluster.local)  
    * uris path segment (/forge-system/sa/... → /\<namespace\>/sa/...)  
* Extra DNS names  
  * For carbide-api, you already have carbide-api.forge as an additional DNS SAN; you can add more if you front it with another internal hostname or external DNS.

#### Postgres cluster patch

| \- patch: |-    apiVersion: acid.zalan.do/v1    kind: postgresql    metadata:      name: forge-pg-cluster  target:    group: acid.zalan.do    version: v1    kind: postgresql    name: forge-pg-cluster |
| :---- |

This is effectively a no-op patch that ensures that the postgresql CR named forge-pg-cluster is under this kustomization. It can also be extended in overlays to adjust configuration.

### **External Services (external-services.yaml)**

This file defines all “northbound” LoadBalancer services that expose key components externally. They are **network plumbing only**

* **Purpose:** Expose carbide-dhcp on UDP/67 to external networks.  
* **Key knobs:**  
  * type: LoadBalancer – could be changed to NodePort or ClusterIP if environment doesn’t support LB.  
  * internalTrafficPolicy: Local – ensures traffic stays local to node endpoints (useful for preserving source IP).  
  * port/targetPort – change if DHCP is proxied or listening on a different port.  
  * selector – depends on labels set in carbide-dhcp deployment.

#### NTP (three instances)

Services: carbide-ntp-external-0, carbide-ntp-external-1, carbide-ntp-external-2.

* **Purpose:** Expose NTP services (per-instance) on UDP/123; each LB points to a specific StatefulSet pod (carbide-ntp-0/-1/-2).

* **Knobs:**  
  * Number of instances: add/remove services if you change replicas in the NTP StatefulSet.  
  * externalTrafficPolicy: Local – typically retained so NTP replies originate from the node that received the request.  
  * labels.app.kubernetes.io/part-of – purely informational, adjust naming if you change instance scheme.  
  * selector – must match pod names and labels.

#### DNS (UDP & TCP, 2 instances)

Pairs:

* carbide-dns-external-udp-0 / carbide-dns-external-tcp-0  
* carbide-dns-external-udp-1 / carbide-dns-external-tcp-1

**Purpose:** Expose DNS on port 53 (UDP \+ TCP) for two HA instances.

**Metallb annotation:**

* metallb.universe.tf/allow-shared-ip: "carbide-dns-instance-0"  
   Allows multiple services (UDP \+ TCP) to **share the same IP** in MetalLB.

Knobs:

* Number of DNS instances – add more \-N services if you increase StatefulSet replicas.  
* metallb.universe.tf/allow-shared-ip – change the key name to group different service pairs sharing IPs.  
* Protocols – you can drop TCP services if your environment truly doesn’t need TCP DNS (not recommended).

#### PXE (HTTP boot)

Two services:

* carbide-pxe-external (port 8080\)  
* carbide-pxe-external-80 (port 80 → targetPort 8080\)

**Purpose:** Expose PXE HTTP endpoints used for boot artifacts; one on 8080, one on 80 mapping to backend 8080\.

**Knobs:**

* Pick which external port(s) to expose (8080 vs 80).  
* allow-shared-ip annotation groups these two to share a MetalLB IP.  
* externalTrafficPolicy: Local – preserve node semantics.

#### Carbide API external

**Purpose:** Expose carbide-api over TLS on port 443, forwarding to gRPC port 1079 on the pod.

**Knobs:**

* port: 443 – can be changed if you prefer a nonstandard external port.  
* targetPort: 1079 – must match what carbide-api listens on (configurable in its config).  
* externalTrafficPolicy / internalTrafficPolicy – both Local; adjust only if you have specific cross-node load-balancing needs.

#### SSH Console RS external

**Purpose:** Expose carbide-ssh-console-rs pods externally on SSH port 22\.

**Knobs:**

* namespace – explicitly set here to forge-system (even though kustomization namespace already does that).  
* External port – can be changed to another port (e.g., 2222\) if port 22 is restricted at the LB layer.  
* selector – must match ssh-console deployment label.

#### FRRouting (BGP) external

Three services: frrouting-0, frrouting-1, frrouting-2.

**Purpose:** Expose BGP port 179 for each FRR instance (per StatefulSet pod).

**Knobs:**

* Number of FRR instances – add/remove services if you scale the StatefulSet.  
* Ports – BGP is TCP/179, but you could front with different external port if required (e.g., firewall constraints).  
* externalTrafficPolicy: Local – typically good for BGP peering.

#### Nginx external

**Purpose:** Expose nginx (likely a UI / reverse proxy) on port 80\.

**Knobs:**

* External / internal ports.  
* TLS offload (you could front it with a TLS-terminating LB and change the targetPort if nginx runs on 8080 or 8443, etc.).

### **7.12.1 How to Customize Carbide** 

In order to customize Carbide for your site you have to edit the following files:

* deploy/kustomization.yaml  
* deploy/files/carbide-api/admin\_root\_cert\_pem  
* deploy/files/carbide-api/carbide-api-site-config.toml  
* deploy/files/unbound/forwarders.conf  
* deploy/files/unbound/local\_data.conf  
* deploy/files/frr.conf-0  
* deploy/files/frr.conf-1  
* deploy/files/frr.conf-2  
* deploy/files/kea\_config.json

These configuration files are supplied with template variables. This section describes how to customize these files for your site by replacing them with actual values.

Example values used in this section are for a site with 3 site controller nodes and has the following IP plan:

| Example IP Plan |  |
| ----- | ----- |
| **Name** | **Value** |
| ASN Range  | 4268033000 \- 4268033999 |
| Dpu Loopback Pool | 10.45.217.128/25 |
| Admin Network Pool   | 10.45.217.0/25 |
| Service Vip Pool | 10.45.218.32/27 |
| Control Plane Pool | 10.45.218.0/29 |
| Tenant Overlay(s) | 10.45.216.0/24 |
| Unbound Forward Addresses  | 10.45.33.100 10.45.33.102 |
| Control Plane IPMI Pools  | 10.45.220.192/29 10.45.220.200/29 10.45.220.208/29  |
| Managed Host IPMI Pools | 10.45.220.128/26 10.45.220.0/26 10.45.221.0/25 10.45.220.64/26 10.45.221.128/25 |

#### Site and Domain Names and Certificate

Each Carbide site should have a unique name and DNS domain name. These are the variables that define them.   
 

| ENVIORNMENT\_NAME | Site’s name.  Example: demo1 |
| :---- | :---- |
| SITE\_DOMAIN\_NAME | Site’s domain name. You should be able to access the site's admin interface via https://api-\<ENVIRONMENT\_NAME\>.\<SITE\_DOMAIN\_NAME\>.  It may be necessary to add DNS entries for resolving api-\<ENVIRONMENT\_NAME\>.\<SITE\_DOMAIN\_NAME\>. Example:  [example.com](http://example.com)  |
| ADMIN\_ROOT\_CERT\_PEM | Generate a CA chain and place the cert in the following location files/carbide-api/admin\_root\_cert\_pem This would be used to authenticate forge admin cli |

#### Assign IP Addresses for Services

The following variables are IP addresses for the services used by carbide. These addresses are derived from the Service Vip Pool discussed previously. Typically, this pool is a /27 segment that gives 32 IP addresses. There are no restrictions on how IP addresses are assigned from this range.  

| CARBIDE\_API\_EXTERNAL  | This is the IP address for the api service,  It is recommended that you add a DNS entry named  api-ENVIRONMENT\_NAME.SITE\_DOMAIN\_NAME for this IP address.  You may also have to ACL this IP depending on your network configurations.  Example value:  10.45.218.49 |
| :---- | :---- |
| CARBIDE\_DHCP\_EXTERNAL | This is the IP address used by DHCP service in carbide. Example value: 10.45.218.32 |
| CARBIDE\_DNS\_INSTANCE\_0  CARBIDE\_DNS\_INSTANCE\_1   | Carbide DNS services use these two IP addresses. Allocate two contiguous addresses for them.   Example Values: 10.45.218.51  10.45.218.52 |
| CARBIDE\_NGINX  | IP address used by ngnix service in Carbide. Example value: 10.45.218.54 |
| FORGE\_UNBOUND\_EXTERNAL\_IP | This is for the unbound service IP address. Example value: 10.45.218.46 |
| CARBIDE\_NTP\_SERVERS\_0 CARBIDE\_NTP\_SERVERS\_1 CARBIDE\_NTP\_SERVERS\_2 | Carbide has three NTP services and these are the addresses used by them. Example values: 10.45.218.37 10.45.218.38 10.45.218.39  |
| CARBIDE\_PXE  | IP address for carbide PXE service. Example: 10.45.218.34 |
| CARBIDE\_SSH\_CONSOLE\_EXTERNAL | IP address for BMC console access service in Carbide. Example: 10.45.218.36    |
| ENVOY\_EXTERNAL\_SERVICE    | Some carbide services such as ArgoCD server use this as the proxy. Example: 10.45.218.50    |
| FRR\_ROUTING\_0\_EXTERNAL FRR\_ROUTING\_1\_EXTERNAL FRR\_ROUTING\_2\_EXTERNAL  | These are IP addresses assigned to frr service. Example: 10.45.218.35    10.45.218.40 10.45.218.41    |


#### Variables in carbide-api-site-config.toml

Most of the variables in this file are already described in previous sections. This section covers the rest.

| ADMIN\_NETWORK\_IP\_POOL | Admin pool discussed in section \<todo\> Example value: 10.45.217.0/25 |
| :---- | :---- |
| ADMIN\_NETWORK\_GATEWAY\_IP | Gateway IP of admin pool. Usually it is the first IP address in the pool Example value: 10.45.217.1 |
| SITE\_FABRIC\_PREFIX\_1 | This is the first tenant overlay network pool. If there are more than one then they must be added as SITE\_FABRIC\_PREFIX\_2 and so on. SITE\_FABRIC\_PREFIX\_1 \= 10.45.216.0/24 |
| CONTROL\_PLANE\_IPMI\_POOL\_1 | This is the first IPMI pool for site controller nodes. If there are multiple pools they must be included as CONTROL\_PLANE\_IPMI\_POOL\_2 and so on.  Example: CONTROL\_PLANE\_IPMI\_POOL\_1 \= 10.45.220.192/29  CONTROL\_PLANE\_IPMI\_POOL\_2 \= 10.45.220.200/29  CONTROL\_PLANE\_IPMI\_POOL\_3 \= 10.45.220.208/29 |
| MANAGED\_HOST\_IPMI\_POOL\_1 | This is the first IPMI pool for managed hosts. If there are multiple pools they must be included as MANAGED\_HOST\_IPMI\_POOL\_2 and so on.   |
| MANAGED\_HOST\_IPMI\_NETWORK\_1 | Network name of the first managed host IPMI pool.  Each managed host IPMI pool must be given a unique name in *carbide-api-site-config.toml*. There are no restrictions on the name as long as they are unique. Typically switch names are used. If there are multiple pools additional   |
| MANAGED\_HOST\_IPMI\_POOL\_1\_GATEWAY\_IP | Gateway IP of the first managed host IPMI pool.  |
| ADMIN\_ROOT\_CERT\_PEM |  |
| ENVIORNMENT\_NAME |  |
| SITE\_DOMAIN\_NAME |  |

#### Carbide Components and their version tags

These are the components used by carbide:

* nvmetal-carbide  
* boot-artifacts-aarch64  
* boot-artifacts-x86\_64  
* frr  
* machine\_validation

It is assumed that these components are already built and hosted in a registry that is accessible to site controller plane Kubernetes cluster. The credentials to the registry must be available in a secret named \<todo\>

| CARBIDE\_REGISTRY\_PATH | URL to the registry that hosts all carbide components listed above. We assume that all components share this value. If that is not the case you need to manually enter the values in file *deploy/kustomization.yaml*. Example:   nvcr.io/0837451325059433/carbide-dev/ |
| :---- | :---- |
| CARBIDE\_TAG | Version of component **nvmetal-carbide**. Example:  v2025.11.21-rc2-0-0-g7b846fa32 |
| BOOT\_ARTIFACTS\_AARCH64\_TAG | Version of component **boot-artifacts-aarch64**. Example: v2025.11.21-rc2-0-0-g7b846fa32 |
| BOOT\_ARTIFACTS\_X86\_TAG | Version of component **boot-artifacts-x86\_64** Example: v2025.11.21-rc2-0-0-g7b846fa32 |
| MACHINE\_VALIDATION\_TAG | Version of component **machine\_validation**. Example: v2025.11.21-rc2-0-0-g7b846fa32 |
| FRR\_TAG | Frr component version Example: 8.5.0 |

## **7.13 SA Enablement \- Carbide bring up**

Requirements:

* All common services running  
* Vault running and unsealed

### **Terraform**

We need to add proper policies to the cluster using terraform and place the correct certificate in vault. This work has been abstracted away with a script in carbide-external repository.

* Please pull the latest main in carbide-external repository.  
* Make sure you have access to the cluster  
  * Run ssh tunnel command in the background o another terminal  
    ssh \-D 8888 \-N renojump 

* Switch back to repo and make sure you kubectx is pointed to the correct cluster  
  export KUBECONFIG=/PATH/TO/SITE/kubeconfig 

* Terraform might complain about kubeconfig certificate since it looks at   
  \~/.kube/config   
  Make sure to to backup your default config. Then copy over  
  cp /PATH/TO/SITE/kubeconfig  \~/.kube/config 

* Now you can run the terraform script  
  ./manifests/setup\_terraform.sh dev6 

### **Image Pull Secret**

* Since we are still using internal NVIDIA images we need NVCR auth token to pull these images. Export in an env variable  
  export NVCR\_AUTH\_TOKEN=XXX   
* Then create the secret:

| kubectl create secret generic imagepullsecret \\      \--type=kubernetes.io/dockerconfigjson \\      \-n default \\      \--from-literal=.dockerconfigjson="{\\"auths\\":{\\"nvcr.io\\":{\\"auth\\":\\"$NVCR\_AUTH\_TOKEN\\"}}}" |
| :---- |


* 

There is already a branch in the carbide repository called sa-enablement\_dev8.  
Navigate to deploy directory in repo root. It should have the following structure:

| deploy git:(sa-enablement\_dev8) tree \-L 1.├── carbide-base├── carbide-unbound-base├── components├── files├── forge-system├── frrouting-base└── kustomization.yaml |
| :---- |

Run the following command to deploy call carbide code components 

| kustomize build . \--enable-helm \--enable-alpha-plugins \--enable-exec | kubectl apply \-f \- |
| :---- |

After all resources are properly provisioned you should see the following pods

| kubectl \-n forge-system get pods                             NAME                                       READY   STATUS      RESTARTS   AGEcarbide-api-6bfdb9c56-lrfgf                11/11   Running     0          4h22mcarbide-api-migrate-4btcv                  0/1     Completed   0          33hcarbide-dhcp-786fd6ccd6-m82rp              1/1     Running     0          33hcarbide-dns-0                              1/1     Running     0          33hcarbide-dns-1                              1/1     Running     0          33hcarbide-hardware-health-56c9789bcf-bsr7q   1/1     Running     0          33hcarbide-ntp-0                              1/1     Running     0          33hcarbide-ntp-1                              1/1     Running     0          33hcarbide-ntp-2                              1/1     Running     0          33hcarbide-pxe-594fd46b84-5khz4               5/5     Running     0          33hcarbide-ssh-console-rs-6459f76bc9-r6sck    2/2     Running     0          33hcarbide-unbound-89896d5cd-zf5ph            2/2     Running     0          33hfrrouting-0                                2/2     Running     0          33hfrrouting-1                                2/2     Running     0          33hfrrouting-2                                2/2     Running     0          33hnginx-6947c6b57-bx4w8                      5/5     Running     0          33h |
| :---- |

### **Forge-admin-cli**

We want to point forge-admin-cli to carbide api end point. More specifically carbide-api-external service 

|  kubectl \-n forge-system get svc | grep \-i carbide-api carbide-api                       ClusterIP      10.233.5.9      \<none\>           1079/TCP                 33hcarbide-api-external              LoadBalancer   10.233.7.217    10.180.126.241   443:31310/TCP            33hcarbide-api-metrics               ClusterIP      10.233.54.93    \<none\>           1080/TCP                 33hcarbide-api-profiler              ClusterIP      10.233.2.57     \<none\>           1081/TCP                 33h |
| :---- |

In case of dev8 we already have the DNS setup with \=the following URL:

| https://api-dev8.frg.nvidia.com |
| :---- |

#### Using forge-admin-cli with mTLS

##### Overview

Forge uses a private Certificate Authority (CA) to secure mTLS connections to the Carbide API. The CA lives in Vault under the forgeca PKI engine and is wired up by the Terraform scripts you ran earlier.

The high-level flow is:

1. Inspect the Forge CA in Vault. (you don’t have to follow this)  
2. Decrypt and extract the Forge root CA bundle from the carbide-external repo.  
3. Use that root CA to sign a user client certificate.  
4. Run forge-admin-cli with the correct CA and client cert/key.  
5. Run forge-admin-cli with the correct CA and client cert/key without DNS.  
   Kubernetes port forwarding

##### 1\. Inspecting the Forge CA in Vault (No need to follow this just for information)

First, get the Vault root token:

| \# Get Vault root token and store its valuekubectl \-n vault get secrets vaultroottoken \-o yaml \\  | yq \-r '.data\["token"\]' \\  | base64 \-d && echo |
| :---- |

Connect to one of the Vault pods:

| kubectl \-n vault exec \-it vault-0 \-- sh |
| :---- |

Log in to Vault using the root token (inside the pod):

| vault login \-tls-skip-verify |
| :---- |

List enabled secrets engines and locate the Forge PKI backends (we care about forgeca/):

| vault secrets list \-tls-skip-verify |
| :---- |

Example output:

| Path             Type         Accessor              Description\-\---             \----         \--------              \-----------cubbyhole/       cubbyhole    cubbyhole\_bfa0f96d    per-token private secret storageforgeca-root/    pki          pki\_ef1f0dc2          PKI engine hosting root CAforgeca/         pki          pki\_d1f9c3a7          PKI engine hosting intermediate CA forgeidentity/        identity     identity\_7bb3f555     identity storesecrets/         kv           kv\_7eee8ab3           n/asys/             system       system\_ce32184a       system endpoints used for control, policy and debugging |
| :---- |

You can view the Forge CA certificate with:

| vault read \-tls-skip-verify \-field=certificate forgeca/cert/ca |
| :---- |

##### 2\. Decrypting the Forge Root CA Bundle

Switch to carbide-external repository

The Forge root CA bundle is stored (encrypted with SOPS) in the carbide-external repository:

| components/carbide-certificate/secrets/forge-root.pem.enc |
| :---- |

This CA bundle is used to sign intermediate certificates, which in turn sign the client certificates used for mTLS connections.

Make sure you use the correct SOPS key:

| export SOPS\_AGE\_KEY\_FILE=\~/.config/sops/age/carbide-external.txt |
| :---- |

Decrypt the bundle:

| \# Decrypt from SOPS into a local bundlesops \-d components/carbide-certificate/secrets/forge-root.pem.enc \> forge-root-bundle.pem |
| :---- |

Extract just the certificate:

| awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/' forge-root-bundle.pem \> forge-root-ca.crt |
| :---- |

Extract just the private key:

| awk '/BEGIN PRIVATE KEY/,/END PRIVATE KEY/' forge-root-bundle.pem \> forge-root-ca.key |
| :---- |

At this point you should have:

* forge-root-ca.crt – root CA certificate  
* forge-root-ca.key – root CA private key

##### 3\. Creating a Client Certificate for forge-admin-cli

Generate a new private key and CSR for your user (example: mnoori):

| openssl req \-new \\  \-newkey rsa:2048 \\  \-nodes \\  \-keyout mnoori.key \\  \-out mnoori.csr \\  \-subj "/O=NGC Forge/OU=swngc-forge-admins/CN=mnoori" |
| :---- |

Sign the CSR with the Forge root CA:

| openssl x509 \-req \\  \-in mnoori.csr \\  \-CA forge-root-ca.crt \\  \-CAkey forge-root-ca.key \\  \-CAcreateserial \\  \-out mnoori.crt \\  \-days 365 \\  \-extfile \<(cat \<\<EOFbasicConstraints=CA:FALSEkeyUsage \= digitalSignature, keyEnciphermentextendedKeyUsage \= clientAuth,serverAuthEOF) |
| :---- |

You should now have the following files:

* forge-root-ca.crt   
* mnoori.crt – client certificate (named after the username; feel free to change the filename)  
* mnoori.key – client private key

##### 4\. Running forge-admin-cli

From the directory containing the certificate files, run:

| docker run \\  \-v "$(pwd)":/certificates/ \\  nvcr.io/nvidian/nvforge-devel/forge-admin-cli:latest \\    \--forge-root-ca-path /certificates/forge-root-ca.crt \\    \--client-cert-path /certificates/mnoori.crt \\    \--client-key-path /certificates/mnoori.key \\    \-c https://api-dev8.frg.nvidia.com/ instance show |
| :---- |

Example output:

| \+----+-----------+-----------+-------------+----------------+---------------+-------------+--------+| Id | MachineId | TenantOrg | TenantState | InstanceTypeId | ConfigsSynced | IPAddresses | Labels |\+====+===========+===========+=============+================+===============+=============+========+\+----+-----------+-----------+-------------+----------------+---------------+-------------+--------+ |
| :---- |

#####  4\. Running forge-admin-cli (without DNS Kubernetes port forwarding)

Port forward the service

| kubectl \-n forge-system port-forward service/carbide-api-external \--address 0.0.0.0 8181:443 |
| :---- |

Since the cert on carbide-api is issued for api-dev8.frg.nvidia.com (or a SAN that includes it) we need to use the **same hostname** (api-dev8.frg.nvidia.com) so TLS is happy, but point that name at your local port-forward from inside the container

| docker run \\ \--add-host=api-dev8.frg.nvidia.com:host-gateway \\ \-v "$(pwd)":/certificates/ \\ nvcr.io/nvidian/nvforge-devel/forge-admin-cli:latest \\   \--forge-root-ca-path /certificates/forge-root-ca.crt \\   \--client-cert-path /certificates/mnoori.crt \\   \--client-key-path /certificates/mnoori.key \\   \-c https://api-dev8.frg.nvidia.com:8181/ instance show |
| :---- |

## **7.14 Elektra Site Agent (namespace elektra-site-agent)**

### **7.14.1 Role & lifecycle**

Elektra is the **site‑side agent** that:

* Connects to Temporal (cloud \+ site workers) and runs “site” workflows.  
* Talks to **Carbide** over mTLS gRPC to orchestrate hardware operations.  
* Reads/writes to a **site‑local PostgreSQL** database for persistent state.  
* Uses **cert‑manager \+ Vault** to obtain a SPIFFE‑style client cert and trust the Forge root CA.  
* Is bootstrapped **per site** using an OTP \+ CA bundle \+ credentials delivered by the cloud side (cloud‑site‑manager).

Workload shape:

* StatefulSet elektra-site-agent  
  * replicas: 3  
  * serviceAccount: site-agent  
* Services:  
  * elektra-headless (headless, metrics on 2112\)  
  * elektra (ClusterIP, metrics on 2112\)  
* Metrics: port **2112** (scraped by Prometheus via ServiceMonitor if present).

---

### 

### 

### **7.14.2 What must exist before you deploy Elektra**

Before applying the Elektra Kustomize overlays, your cluster must already have:

#### PostgreSQL

* A database reachable from elektra-site-agent namespace with:  
  * DB\_HOST, DB\_PORT, DB\_NAME encoded in ConfigMap elektra-database-config.  
* A Secret referenced as **elektra-site-agent.elektra.forge-pg-cluster.credentials** (or your renamed equivalent) with keys:  
  * username  
  * password  
     Typically this comes from Vault via ESO → Secret.

#### Temporal

* A reachable frontend endpoint, e.g.:  
  * temporal-frontend-headless.temporal.svc.cluster.local:7233 (in‑cluster), or  
  * site.server.temporal.\<your-domain\>:7233 (if you front Temporal with a gateway).  
* Namespaces:  
  * cloud  
  * site  
  * A **per‑site namespace** that will equal cluster\_id / SITE\_UUID (created later by create-site-in-db.sh).

If enable\_tls=true in Elektra, you also need a way to obtain an mTLS client certificate and CA bundle for the site (wired into the temporal-cert Secret later).

#### Carbide core

* A working carbide‑api endpoint, e.g.:  
  * carbide-api.forge-system.svc.cluster.local:1079  
* If carbide\_grpc=1 and enable\_tls=true, ensure Elektra has a valid client certificate and trusts the Carbide server’s CA (via SPIFFE client cert from Vault/cert‑manager).

#### Vault \+ cert‑manager

* A ClusterIssuer for site‑agent mTLS, e.g. **vault-forge-issuer**.  
* A CertificateRequestPolicy that allows the Elektra namespace to request a client cert with a SPIFFE URI like:  
  * spiffe://forge.local/elektra-site-agent/sa/\<SA\_NAME\>

The grpc‑client overlay already defines:

* CertificateRequestPolicy site-agent-approver-policy  
* ClusterRole/ClusterRoleBinding giving cert-manager permission to **use** that policy  
* A Certificate grpc-client-cert issuing Secret elektra-site-agent-grpc-client-cert via vault-forge-issuer.

You just need to ensure your **ClusterIssuer name / SPIFFE root** match your PKI.

---

### 

### 

### **7.14.3 Configuration knobs (ConfigMaps → env)**

The **base** config lives in base/files/config.properties and is turned into ConfigMap elektra-config-map. That, plus elektra-database-config, feeds almost all Elektra environment variables via the StatefulSet.

Key knobs SAs/customers must understand:

#### Temporal & TLS

* temporal\_server  
   Hostname of the site Temporal gateway Elektra connects to, e.g.:  
  * site.server.temporal.nvidia.com (external name), or  
  * temporal-frontend-headless.temporal.svc.cluster.local (in‑cluster).  
* temporal\_cert\_path  
   Filesystem path where the Temporal mTLS secret is mounted, typically:  
  * /var/secrets/temporal/certs  
* Must match the temporal-auth volume mount in the StatefulSet (which projects the temporal-cert Secret).  
* enable\_tls (true/false)  
   Whether Elektra uses mTLS for Temporal.  
* temporal\_inventory\_schedule  
   Cron‑style string for periodic inventory workflows (e.g. @every 3m).  
* temporal\_subscribe\_queue, temporal\_publish\_queue  
   Task queues used for inbound/outbound workflows (cloud‑workflow must match these).

#### Site identity & Temporal namespace

* cluster\_id  
   The **site UUID**. Used as:  
  * CLUSTER\_ID env  
  * TEMPORAL\_SUBSCRIBE\_NAMESPACE (via cluster\_id)  
* It must match:  
  * The SITE\_UUID used in site.sql and in DB rows,  
  * The Temporal namespace created for the site, and  
  * The Site CR created by cloud‑site‑manager.

In the overlay, you will explicitly set cluster\_id=\<SITE\_UUID\> and temporal\_subscribe\_namespace=\<SITE\_UUID\>; see 7.4.5/7.4.6.

#### Database (elektra-database-config)

elektra-database-config is generated by the base kustomization and contains:

* DB\_HOST – host for the site DB (default: forge-pg-cluster.postgres.svc.cluster.local)  
* DB\_PORT – port (default: 5432\)  
* DB\_NAME – DB name (default: elektra)  
* SECRET\_REF – name of the DB credentials Secret in your environment (defaults to elektra-site-agent.elektra.forge-pg-cluster.credentials.postgresql.acid.zalan.do)

The StatefulSet expects a Secret named **elektra-site-agent.elektra.forge-pg-cluster.credentials** (without the .postgresql.acid… suffix) for DB\_USER and DB\_PASSWORD. Adjust the Secret name and references consistently if you rename them.

#### Carbide integration

* carbide\_address – address of Carbide’s gRPC endpoint (default: carbide-api.forge-system.svc.cluster.local:1079).  
* carbide\_grpc / carbide\_grpc\_skip\_auth – control whether Elektra uses gRPC to talk to Carbide and whether server‑side identity checking is enforced.

Typically for a hardened deployment you eventually set:

* carbide\_grpc=1  
* carbide\_grpc\_skip\_auth=false

after your SPIFFE/mTLS wiring is stable.

#### Operational knobs

* dev\_mode, enable\_debug, clientset\_log  
* metrics\_port (default 2112\)  
* site\_workflow\_version, cloud\_workflow\_version – semantic tags that must match the versions of your site/cloud workflows in Temporal.  
* upgrade\_cron\_schedule, upgrade\_frequency\_week\_nums, upgrade\_batches – govern when/how upgrade workflows run.

If you integrate with Lightstep/OTel, set these and ensure an appropriate otel-lightstep-access Secret exists. Otherwise, leave them unset and/or disable the OTEL integration paths in Elektra.

---

### **7.14.4 Certificates & gRPC client identity**

#### gRPC client cert (SPIFFE)

The grpc‑client overlay defines:

* CertificateRequestPolicy site-agent-approver-policy  
  * issuerRef → ClusterIssuer vault-forge-issuer  
  * Scoped to namespace: elektra-site-agent  
  * Allows DNS SANs:  
    * \*.svc, \*.cluster.local, \*.svc.cluster.local  
  * Allows URIs with prefix:  
    * spiffe://forge.local/elektra-site-agent/sa/\*  
* Certificate grpc-client-cert  
  * secretName: elektra-site-agent-grpc-client-cert  
  * dnsNames: elektra.elektra-site-agent.svc.cluster.local  
  * uris: spiffe://forge.local/elektra-site-agent/sa/elektra-site-agent

The StatefulSet mounts this Secret as:

* /etc/carbide (volume spiffe).

#### Customer knobs

If your PKI differs, update:

* selector.issuerRef.name in the CertificateRequestPolicy,  
* spec.issuerRef.name, and SAN/URI values in grpc-client-cert,  
* The secretName used by the spiffe volume in the StatefulSet.

Make sure Vault \+ cert‑manager policies allow the **site-agent ServiceAccount** to request that cert. Elektra won’t establish mTLS correctly with Carbide/other services if this cert is missing and enable\_tls=true.

#### Dynamic secrets used by Elektra

1. **Database credentials Secret**  
   * Name: elektra-site-agent.elektra.\<your-db-cluster\>.credentials (or equivalent).  
   * Keys: username, password.  
   * Source: typically Vault → ESO → Secret.  
   * Used by envs DB\_USER and DB\_PASSWORD.

2. **Site registration Secret bootstrap-info** – **runtime only**  
    Not in static manifests. Expected keys:  
   * site-uuid – must equal cluster\_id.  
   * otp – OTP returned by cloud‑site‑manager.  
   * creds-url – URL Elektra calls to fetch long‑lived site credentials.  
   * cacert – CA chain to trust that URL.  
3. Mounted at /etc/sitereg (one file per key). This is created by setup-site-bootstrap.sh (see 7.4.5).  
4. **Temporal certificate Secret temporal-cert** – **runtime**  
    Base manifests include a placeholder Secret with AutoGenerated values to satisfy tooling. Elektra expects keys:  
   * otp  
   * cacertificate  
   * certificate  
   * key  
5. Mounted via projected volume temporal-auth at /var/secrets/temporal/certs, which must match temporal\_cert\_path.

    In a real deployment, your bootstrap flow (or operator) replaces the placeholder with real mTLS material. setup-site-bootstrap.sh creates an empty scaffold (values are blank) so you can wire in real certs later.

---

### **7.14.5 Site bootstrap workflow (helper scripts)**

To avoid hand‑editing UUIDs, SQL, and secrets, the Elektra repo includes three helper scripts in elektra-site-agent/deploy/. SAs should run them **in order**.

All scripts default to the current kube context; you can override via KUBECTL\_CONTEXT=\<context\>.

#### 1\) gen-site-sql.sh – generate site.sql \+ SITE\_UUID

From elektra-site-agent/deploy:

| ./gen-site-sql.sh |
| :---- |

What it does:

* Generates a **SITE\_UUID** (or uses SITE\_UUID env if set).  
* Uses the directory name as **site name** (or SITE\_NAME env).  
* Writes a site.sql file (next to the script) containing two inserts:  
  * one into infrastructure\_provider,  
  * one into site with id \= SITE\_UUID.

Typical output:

| 🔖  Site name : deploy🔑  SITE\_UUID : 12bd0e80-3382-40d6-82d0-0b20dc14be11📌  First-state log: SITE\_UUID=12bd0e80-3382-40d6-82d0-0b20dc14be11✅  Generated .../deploy/site.sql |
| :---- |

#### 2\) create-site-in-db.sh – insert site rows \+ Temporal namespace

Then:

| ./create-site-in-db.sh\# or ./create-site-in-db.sh /path/to/site.sql |
| :---- |

What it does:

* Extracts SITE\_UUID from site.sql.  
* Reads the cloud DB name from ConfigMap cloud-db-config in namespace cloud-db.  
* Locates the master Postgres pod in namespace postgres.  
* If no site row for this SITE\_UUID exists, pipes site.sql into psql to insert it.  
* Uses the Temporal admintools pod to ensure a namespace named SITE\_UUID exists  
   (tctl \--ns \<SITE\_UUID\> namespace register if needed).

Typical output:

| Using SITE\_UUID=12bd0e80-3382-40d6-82d0-0b20dc14be11⏳  Checking if site rows already exist...Inserting site data into Postgres...INSERT 0 1INSERT 0 1⏳  Ensuring Temporal namespace 12bd0e80-3382-40d6-82d0-0b20dc14be11 exists...Namespace 12bd0e80-3382-40d6-82d0-0b20dc14be11 successfully registered.✓  Temporal namespace 12bd0e80-3382-40d6-82d0-0b20dc14be11 registered.🎉  Site DB rows & Temporal namespace ensured for SITE\_UUID=12bd0e80-3382-40d6-82d0-0b20dc14be11. |
| :---- |

#### 3\) setup-site-bootstrap.sh – create site CR \+ secrets

Finally:

| ./setup-site-bootstrap.sh\# Optional: SITE\_UUID=\<existing\> ./setup-site-bootstrap.sh |
| :---- |

What it does:

1. **Determines SITE\_UUID**  
   * From SITE\_UUID env or by parsing site.sql.  
2. **Ensures nettools-pod is running**  
   * kubectl apply \-f nettools-pod.yaml, waits for phase Running.  
3. **Creates/ensures the Site CR via cloud‑site‑manager**  
   * Calls https://sitemgr.cloud-site-manager:8100/v1/site from nettools-pod with JSON payload containing siteuuid=\<SITE\_UUID\>.  
4. **Fetches the CA cert from credsmgr**  
   * curl https://credsmgr.cert-manager:8000/v1/pki/ca/pem → /tmp/cacert.pem, then copies it locally.  
5. **Reads the OTP from the Site CR**  
   * kubectl get site site-\<SITE\_UUID\> \-n cloud-site-manager \-o jsonpath='{.status.otp.passcode}'  
6. **Ensures namespace elektra-site-agent exists**.  
7. **Creates/updates two secrets in elektra-site-agent:**  
   * bootstrap-info:  
     * literals site-uuid, otp, creds-url, and cacert file  
   * temporal-cert:  
     * blank literal values for cacertificate, certificate, key, otp (scaffold only).

Typical output:

| Using SITE\_UUID=12bd0e80-3382-40d6-82d0-0b20dc14be11Ensuring nettools-pod exists...nettools-pod is Running.Creating site via site-manager...sites.forge.nvidia.io "site-12bd0e80-3382-40d6-82d0-0b20dc14be11" already existsFetching CA certificate from credsmgr...Reading OTP from Site CR...OTP=4waJOq-GEETjPTS134HCCw0DIiU=Ensuring namespace elektra-site-agent exists...namespace/elektra-site-agent createdCreating / updating bootstrap-info secret in elektra-site-agent...secret/bootstrap-info createdCreating / updating temporal-cert secret in elektra-site-agent...secret/temporal-cert created🎉  Site-agent bootstrap secrets created. No restart performed. |
| :---- |

At this point:

* DB rows and Temporal namespace exist.  
* Site CR, OTP, CA, and bootstrap-info / temporal-cert secrets exist.  
* You are ready to wire the UUID into the overlay and deploy Elektra.

---

### 

### 

### **7.14.6 Overlay configuration (per‑site UUID) & deployment**

The overlay lives under elektra-site-agent/deploy/overlay/ and imports the base:

| resources:  \- ../baseimages:  \- name: nvcr.io/nvidian/nvforge-devel/forge-elektra    newTag: v2025.10.10-rc1-0configMapGenerator:  \- name: elektra-config-map    behavior: merge    env: config.properties |
| :---- |

The **only fields you must always adjust per site** are in overlay/config.properties:

| cluster\_id=12bd0e80-3382-40d6-82d0-0b20dc14be11temporal\_host=temporal-frontend-headless.temporal.svc.cluster.localtemporal\_port=7233temporal\_publish\_namespace=sitetemporal\_publish\_queue=sitetemporal\_subscribe\_namespace=12bd0e80-3382-40d6-82d0-0b20dc14be11temporal\_subscribe\_queue=sitecarbide\_grpc=2 |
| :---- |

**SA actions:**

1. Set cluster\_id to the **exact SITE\_UUID** printed by gen-site-sql.sh.  
2. Set temporal\_subscribe\_namespace to the **same SITE\_UUID**.  
3. Adjust temporal\_host (and temporal\_port if needed) if your Temporal frontend Service name differs.  
4. Optionally tweak carbide\_grpc / other knobs according to policy.

This ensures:

* Elektra’s CLUSTER\_ID and TEMPORAL\_SUBSCRIBE\_NAMESPACE match the DB/site/Temporal namespace.  
* Elektra connects to the correct Temporal host for this site.

#### Deploy Elektra

With:

* DB Secret \+ elektra-database-config in place,  
* Temporal endpoint \+ namespaces (cloud, site, and \<SITE\_UUID\>) ready,  
* Vault \+ cert‑manager (vault-forge-issuer) configured,  
* bootstrap-info & temporal-cert created via setup-site-bootstrap.sh,  
* overlay/config.properties updated as above,

deploy via:

| kubectl apply \-k ./overlay |
| :---- |

Expected creation:

| namespace/elektra-site-agent configuredserviceaccount/site-agent createdrole.rbac.authorization.k8s.io/site-agent createdclusterrole.rbac.authorization.k8s.io/cert-manager-policy:elektra-site-agent createdrolebinding.rbac.authorization.k8s.io/site-agent createdclusterrolebinding.rbac.authorization.k8s.io/cert-manager-policy:elektra-site-agent createdconfigmap/elektra-config-map createdconfigmap/elektra-database-config createdservice/elektra createdservice/elektra-headless createdstatefulset.apps/elektra-site-agent createdcertificate.cert-manager.io/grpc-client-cert createdcertificaterequestpolicy.policy.cert-manager.io/site-agent-approver-policy created |
| :---- |

Verify pods:

| kubectl \-n elektra-site-agent get pods |
| :---- |

You should eventually see something like:

| elektra-site-agent-0   1/1   Running   ...elektra-site-agent-1   1/1   Running   ...elektra-site-agent-2   1/1   Running   ... |
| :---- |

At this point, the site agent is fully wired:

* Uses **mTLS** & SPIFFE to talk to Carbide (and optionally other services).  
* Connects to Temporal with:  
  * publish namespace/queue: site / site  
  * subscribe namespace: \<SITE\_UUID\> / queue site  
* Reads its per‑site identity and credentials from the DB, Temporal, bootstrap-info, and temporal-cert secrets.

### 

## 

## **8\) Networking checklist (Contour/Envoy \+ LB)**

* **Contour/Envoy** (reference: Contour **1.25.2**, Envoy **1.26.4**)  
  * Create Ingress/HTTPProxy entries for any external UIs/APIs you expose.  
  * Use **TLS** via your cert‑manager Issuer (secretName: \<YOUR\_CERT\_SECRET\>).

* **LoadBalancer**  
  * If on‑prem: configure **MetalLB v0.14.5** with your own IPAddressPool CIDRs and **(optionally) BGP** peerings.  
  * If on cloud: use the provider LB and map Service type LoadBalancer on required ports (e.g., cloud-api on 80/9360, carbide-api on 443).

## **9\) Host Ingestion**

After carbide is up and running you can begin ingesting managed hosts. Before you begin that make sure that:

* You have the **forge-admin-cli** command available   
  You can compile it from sources or you can use pre compiled binary. Another choice is to use a containerized version.  
* You can access the carbide site using admin cli.   
  The api service is running at IP address CARBIDE\_API\_EXTERNAL. It is recommended that you add t  
  Run the command 

  **admin-cli \-c [https://api-](https://api-)\<ENVIRONMENT\_NAME\>.\<SITE\_DOMAIN\_NAME\> mi show**  
* DHCP requests from all managed host IPMI networks have been forwarded to service running in Carbide whose IP address is CARBIDE\_DHCP\_EXTERNAL. THis must be completed by the site's networking team.  
* You have the following information for all hosts that need to be ingested:  
  * Mac address of the host BMC  
  * Chassis serial number   
  * Host BMC user name (typically this is factory default username)  
  * Host BMC password (typically this is factory default password)

Upon first login Carbide requires to change factory default credentials to a new set of site wide defaults. You need to set these new credentials as well. These are the new credentials that need to be set:

1. Host BMC Credential  
2. DPU BMC Credential  
3. Host UEFI password  
4. DPU UEFI password

### **Update site default credentials**

Run these commands to set the default credentials. \<**API-URL\>** is typically the following:  
**https://api-\<ENVIRONMENT\_NAME\>.\<SITE\_DOMAIN\_NAME\>**

### **Update DPU UEFI password**

| https://api-\<ENVIRONMENT\_NAME\>.\<SITE\_DOMAIN\_NAME\> |
| :---- |

### **Update Host UEFI password**

Generate a new password using this command:

| admin-cli \-c \<api-url\> host generate-host-uefi-password |
| :---- |

Run this command to update host uefi password:

| admin-cli \-c \<api-url\> credential add-uefi \--kind=host \--password='x' |
| :---- |

	

### **Host and DPU BMC Password**

| admin-cli \-c \<api-url\> credential add-bmc \--kind=site-wide-root \--password='x' |
| :---- |

### **Approve all machine for ingestion**

You need to configure a rule to automatically approve all machines. Failing to do this will cause these machines to raise health alerts. Run this command to complete this step:

| admin-cli \-c \<api-url\> mb site trusted-machine approve \\\* persist \--pcr-registers="0,3,5,6" |
| :---- |

### 

### **Add Expected Machines Table**

First prepare expected machine JSON file as follows:

| {  "expected\_machines": \[    {      "bmc\_mac\_address": "C4:5A:B1:C8:38:0D",      "bmc\_username": "root",      "bmc\_password": "default-password1",      "chassis\_serial\_number": "SERIAL-1"    },    {      "bmc\_mac\_address": "C4:5A:FF:FF:FF:FF",      "bmc\_username": "root",      "bmc\_password": "default-password2",      "chassis\_serial\_number": "SERIAL-2"    }  \]} |
| :---- |

Only servers listed in this table will be ingested. So you have to include all servers in this file. When the file is ready upload it to the site by this command:

| admin-cli \-c \<api-url\> credential em replace-all \--filename expected\_machines.json |
| :---- |

