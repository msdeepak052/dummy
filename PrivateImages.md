# Private ImagePull
**Kubernetes pulling private container images from a private registry**, the key mechanism is an **image pull secret**.

The production flow is:

```text
Pod
 |
 | image: registry.company.com/myapp:v1
 v
Kubelet
 |
 | needs credentials
 v
imagePullSecrets
 |
 v
Private Registry
 |
 v
Pull image
```

Let's do a complete demo.

---

# 1. What problem are we solving?

Suppose your image is stored in a private registry:

```text
registry.example.com
        |
        +-- platform/myapp:1.0
```

Your Pod:

```yaml
containers:
  - name: myapp
    image: registry.example.com/platform/myapp:1.0
```

If the registry requires authentication, Kubernetes needs credentials.

Without credentials you'll typically see:

```bash
kubectl describe pod myapp
```

and events such as:

```text
ErrImagePull
ImagePullBackOff
```

---

# 2. The basic solution: `imagePullSecret`

Create a Kubernetes Secret containing registry credentials:

```text
Secret
  |
  +-- registry URL
  +-- username
  +-- password/token
```

Then tell the Pod:

```yaml
imagePullSecrets:
  - name: regcred
```

Full architecture:

```text
                    Kubernetes
                        |
                        v
                       Pod
                        |
                 imagePullSecrets
                        |
                        v
                    regcred
                        |
                        v
                  Private Registry
                        |
                        v
                       Image
```

---

# 3. Create the registry secret

For a Docker-compatible private registry:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword'
```

You can also specify an email with older workflows, but it generally isn't required.

Check:

```bash
kubectl get secret regcred
```

You'll see:

```text
NAME       TYPE                             DATA
regcred    kubernetes.io/dockerconfigjson   1
```

The important part is:

```text
TYPE:
kubernetes.io/dockerconfigjson
```

---

# 4. What is actually stored?

The Secret contains Docker registry authentication configuration.

Conceptually:

```json
{
  "auths": {
    "registry.example.com": {
      "username": "myuser",
      "password": "mypassword",
      "auth": "..."
    }
  }
}
```

Kubernetes stores this as a Secret.

Important:

> Kubernetes Secrets are **not automatically encrypted just because the object is a Secret**. Protect RBAC access to Secrets and configure encryption at rest for the cluster where appropriate.

---

# 5. Use the secret in a Pod

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: private-app

spec:

  imagePullSecrets:
    - name: regcred

  containers:
    - name: app
      image: registry.example.com/platform/myapp:1.0
```

Now kubelet can use:

```text
regcred
```

when pulling:

```text
registry.example.com/platform/myapp:1.0
```

---

# 6. Full demo

Create namespace:

```bash
kubectl create namespace demo
```

Create the secret **in that namespace**:

```bash
kubectl create secret docker-registry regcred \
  -n demo \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password='mypassword'
```

Then:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: private-app
  namespace: demo

spec:
  imagePullSecrets:
    - name: regcred

  containers:
    - name: app
      image: registry.example.com/platform/myapp:1.0
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Check:

```bash
kubectl get pod -n demo
```

And:

```bash
kubectl describe pod private-app -n demo
```

Look at:

```text
Events:
```

You should see the image pull succeeding if the registry credentials and image are valid.

---

# 7. Very important: Secret must be in the same namespace

Suppose:

```text
Secret:
regcred
namespace: production
```

and:

```text
Pod:
myapp
namespace: development
```

The Pod cannot use that Secret.

Secrets are namespaced resources.

You need:

```text
production
  |
  +-- regcred
  +-- Pod

development
  |
  +-- regcred
  +-- Pod
```

So this is wrong:

```text
production/regcred
       |
       X
development/myapp
```

---

# 8. Better approach: attach it to a ServiceAccount

Instead of adding this to every Pod:

```yaml
imagePullSecrets:
  - name: regcred
```

you can associate the secret with a ServiceAccount.

Create:

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: app-sa
  namespace: demo

imagePullSecrets:
  - name: regcred
```

Then:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: private-app
  namespace: demo

spec:
  serviceAccountName: app-sa

  containers:
    - name: app
      image: registry.example.com/platform/myapp:1.0
```

Now Pods using:

```text
app-sa
```

can inherit the image pull secret configuration.

Architecture:

```text
Pod
 |
 +-- serviceAccountName: app-sa
                |
                v
          ServiceAccount
                |
                +-- imagePullSecrets
                        |
                        v
                     regcred
                        |
                        v
                 Private Registry
```

This is cleaner when many workloads in a namespace use the same registry.

---

# 9. Deployment example

Normally you aren't creating Pods directly.

You use a Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: private-app
  namespace: demo

spec:
  replicas: 3

  selector:
    matchLabels:
      app: private-app

  template:
    metadata:
      labels:
        app: private-app

    spec:
      imagePullSecrets:
        - name: regcred

      containers:
        - name: app
          image: registry.example.com/platform/myapp:1.0
          ports:
            - containerPort: 8080
```

Notice where `imagePullSecrets` is:

```text
Deployment
  |
  v
spec.template.spec
       |
       +-- imagePullSecrets
```

**Not**:

```yaml
spec:
  imagePullSecrets:
```

for the Deployment itself.

It belongs to the **Pod template**.

---

# 10. Docker Hub private repository

For a private Docker Hub repository:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password='MY_TOKEN'
```

Then:

```yaml
imagePullSecrets:
  - name: regcred
```

Use a **Docker Hub access token** rather than your normal password where supported.

---

# 11. AWS ECR

If you're using EKS, this is especially important.

For Amazon ECR, you often **don't need to manually create a Kubernetes `docker-registry` Secret** when using the standard EKS/IAM approach.

Instead:

```text
Pod
 |
 v
ECR
 |
 v
AWS IAM
```

The node or workload obtains appropriate AWS credentials and the container runtime authenticates to ECR.

For EKS, the exact setup depends on whether you're using:

```text
Node IAM role
IRSA
EKS Pod Identity
```

For modern EKS workloads, **EKS Pod Identity** or appropriate IAM configuration is generally preferable to putting long-lived registry credentials into Kubernetes Secrets.

Conceptually:

```text
                   EKS
                    |
                   Pod
                    |
                    v
                  ECR
                    |
                    v
               AWS IAM auth
                    |
                    v
                 Pull image
```

---

# 12. GCR / Artifact Registry

For Google Artifact Registry, you can similarly use registry authentication through a Kubernetes Secret, but in GKE you can often use the platform's workload identity mechanisms rather than distributing static registry credentials.

General pattern:

```text
Pod
 |
 v
Artifact Registry
 |
 v
Cloud identity
 |
 v
Pull image
```

The exact implementation depends on your GKE identity setup.

---

# 13. Azure Container Registry

For Azure Container Registry:

```text
Pod
 |
 v
ACR
 |
 v
Azure identity/authentication
 |
 v
Pull image
```

You can use a Kubernetes image-pull Secret, or in Azure-managed Kubernetes environments use identity-based approaches where supported.

---

# 14. Private registry + Helm

Since we just discussed Helm, here's how you'd normally parameterize it.

`values.yaml`:

```yaml
image:
  repository: registry.example.com/platform/myapp
  tag: "1.0"

imagePullSecrets:
  - name: regcred
```

Deployment template:

```yaml
spec:
  template:
    spec:

      imagePullSecrets:
        {{- toYaml .Values.imagePullSecrets | nindent 8 }}

      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Then:

```bash
helm upgrade --install myapp ./chart \
  -n production \
  --create-namespace \
  -f values-prod.yaml
```

---

# 15. Better Helm pattern

A common chart pattern is:

```yaml
imagePullSecrets: []
```

in `values.yaml`.

Then:

```yaml
spec:
  template:
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

If:

```yaml
imagePullSecrets:
  - name: regcred
```

then Helm renders:

```yaml
imagePullSecrets:
  - name: regcred
```

If it's empty:

```yaml
imagePullSecrets: []
```

then nothing is rendered.

---

# 16. Don't put registry passwords in `values.yaml`

Avoid:

```yaml
registry:
  username: myuser
  password: mypassword
```

in Git.

Even if Helm encodes the resulting Secret as base64:

```text
base64 ≠ encryption
```

Instead consider:

```text
External Secrets Operator
AWS Secrets Manager
Azure Key Vault
Google Secret Manager
HashiCorp Vault
Sealed Secrets
SOPS
```

For example:

```text
AWS Secrets Manager
       |
       v
External Secrets Operator
       |
       v
Kubernetes Secret
       |
       v
Pod
       |
       v
Private Registry
```

---

# 17. Production architecture I would recommend

For a cloud-managed Kubernetes platform:

```text
                   Developer
                       |
                       v
                     Git
                       |
                       v
                    Argo CD
                       |
                       v
                  Deployment
                       |
                       v
                      Pod
                       |
                       v
                Private Registry
                       |
                       v
              Cloud IAM / Identity
```

For EKS + ECR specifically:

```text
                 EKS
                  |
                  v
                 Pod
                  |
                  v
                 ECR
                  |
                  v
          IAM authentication
                  |
                  v
             Pull image
```

Prefer **identity-based authentication** where your platform supports it instead of distributing long-lived registry passwords.

---

# 18. Troubleshooting private image pulls

If you get:

```text
ImagePullBackOff
```

don't immediately recreate the Pod.

Run:

```bash
kubectl describe pod <pod-name>
```

Look at:

```text
Events
```

---

### Error: authentication failure

Something like:

```text
unauthorized
authentication required
```

Check:

```bash
kubectl get secret regcred -o yaml
```

and verify:

```text
registry URL
username/token
```

Also ensure the Secret is in the **same namespace**.

---

### Error: image doesn't exist

```text
manifest unknown
```

Check:

```yaml
image: registry.example.com/platform/myapp:1.0
```

Does that exact tag exist?

---

### Error: Secret not found

You may see something like:

```text
secret "regcred" not found
```

Check:

```bash
kubectl get secret regcred -n <namespace>
```

---

### Error: wrong registry

You might have:

```text
Secret:
registry.company.com

Image:
registry.other.com/myapp
```

The credentials don't necessarily apply to the requested registry.

---

# 19. Important interview question

### Q: What is the difference between `imagePullSecrets` and a normal Secret?

A normal Secret:

```text
kind: Secret
```

can contain arbitrary application configuration or credentials.

An image pull Secret generally uses:

```text
type: kubernetes.io/dockerconfigjson
```

and contains registry authentication information used by the kubelet/container runtime when pulling images.

---

# 20. Important interview question

### Q: Where can you configure `imagePullSecrets`?

Primarily on the **Pod spec**:

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

Or associate the image-pull Secret with a ServiceAccount that Pods use.

---

# 21. Important interview question

### Q: Can a Pod use an image pull Secret from another namespace?

**No.**

The Secret must be available in the Pod's namespace.

---

# 22. Important interview question

### Q: Does Kubernetes pull the image using the API server?

Conceptually, no.

The flow is more like:

```text
API Server
   |
   v
Scheduler
   |
   v
Pod assigned to Node
   |
   v
Kubelet
   |
   v
Container Runtime
   |
   v
Private Registry
```

The **container runtime on the node** performs the actual image pull, using credentials made available through Kubernetes configuration.

---

# 23. Complete interview-ready flow

Say you have:

```yaml
image: registry.company.com/payment/api:2.0
```

and:

```yaml
imagePullSecrets:
  - name: regcred
```

The flow is:

```text
kubectl apply
       |
       v
API Server
       |
       v
Deployment
       |
       v
ReplicaSet
       |
       v
Pod
       |
       v
Scheduler assigns Node
       |
       v
Kubelet
       |
       v
Container Runtime
       |
       +---- read imagePullSecret credentials
       |
       v
registry.company.com
       |
       v
Authenticate
       |
       v
Pull payment/api:2.0
       |
       v
Start container
```

---

# 24. The three approaches to remember

### Basic Kubernetes approach

```text
Private Registry
       |
       v
docker-registry Secret
       |
       v
imagePullSecrets
       |
       v
Pod
```

### ServiceAccount approach

```text
ServiceAccount
       |
       v
imagePullSecrets
       |
       v
Multiple Pods
```

### Cloud-native approach

```text
Pod
 |
 v
Cloud Registry
 |
 v
Cloud IAM / Workload Identity
 |
 v
Image pull
```

For **EKS/ECR**, I'd favor the third approach where possible.

---

## Senior DevOps interview answer

If they ask:

> **"How do you configure Kubernetes to pull private images?"**

A strong answer:

> "For a generic private registry, I'd create a `kubernetes.io/dockerconfigjson` Secret using `kubectl create secret docker-registry`, then reference it through `imagePullSecrets` in the Pod template or associate it with the ServiceAccount used by the workloads. The Secret must exist in the same namespace as the Pod. In a managed cloud environment, I'd prefer identity-based authentication where supported—for example, using AWS IAM with ECR on EKS—rather than distributing long-lived registry passwords. If an image pull fails, I'd check `kubectl describe pod` events for authentication failures, incorrect image/tag, missing Secret, or registry connectivity issues."

The mental model is simply:

```text
               PRIVATE IMAGE
                     |
                     v
              Private Registry
                     ^
                     |
               Authentication
                     ^
                     |
             +-------+-------+
             |               |
       imagePullSecret   Cloud IAM
             |               |
             v               v
            Pod             Pod
```

For your **EKS + Argo CD + Helm** setup, the next level to know is how to do this **without storing registry credentials in Git**, using **ECR + EKS Pod Identity/IAM**, and how that differs from `imagePullSecrets`.
