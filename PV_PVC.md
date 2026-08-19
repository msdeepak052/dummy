# Kubernetes Storage

**PV/PVC = manual storage management**
**StorageClass = automatic storage provisioning**

I'll first explain the concepts, then do **two complete demos**:

1. **PV + PVC with manual binding**
2. **StorageClass + PVC with dynamic provisioning**

---

# 1. The problem Kubernetes is solving

Suppose your application has a Pod:

```text
Pod
 |
 | needs 10 Gi of persistent storage
 v
?????
```

If the Pod dies:

```text
Pod X  ---> deleted
```

You don't want its data to disappear.

So Kubernetes separates:

```text
Application
    |
   PVC
    |
   PV
    |
Actual Storage
```

Think of it like:

```text
PVC = "I need storage"

PV = "Here is storage"

StorageClass = "I can create storage for you"
```

---

# 2. PV — PersistentVolume

A **PersistentVolume (PV)** is a piece of storage made available to the Kubernetes cluster.

Example:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  hostPath:
    path: /data/myapp
```

Conceptually:

```text
Kubernetes Cluster
        |
        v
     +------+
     |  PV  |
     | 5 Gi |
     +------+
        |
        v
 /data/myapp
```

The PV represents the **actual storage resource**.

---

# 3. PVC — PersistentVolumeClaim

A PVC is a **request for storage**.

For example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 2Gi
```

The application doesn't normally say:

> "Give me PV `my-pv`."

Instead it says:

> "I need 2Gi of storage."

```text
             PVC
      "I need 2 Gi"
            |
            v
           PV
        "I have 5 Gi"
```

Kubernetes binds the PVC to a suitable PV.

---

# 4. Manual PV/PVC binding

Let's understand this first because **StorageClass is built on top of this concept**.

Suppose we manually create:

```text
PV
5 Gi
```

and then:

```text
PVC
2 Gi
```

Kubernetes sees:

```text
PVC:
Need = 2 Gi
AccessMode = RWO

        ↓

PV:
Capacity = 5 Gi
AccessMode = RWO

        ↓

       BIND

        ↓

PVC ───────── PV
```

The PVC gets:

```text
STATUS: Bound
```

---

# 5. Complete manual PV/PVC demo

We'll create:

```text
Node
 |
 +-- /data/myapp
 |
 +-- PV: my-pv
       |
       +-- PVC: my-pvc
              |
              +-- Pod
```

## Step 1 — Create PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 5Gi

  accessModes:
    - ReadWriteOnce

  persistentVolumeReclaimPolicy: Retain

  hostPath:
    path: /data/myapp
```

Apply:

```bash
kubectl apply -f pv.yaml
```

Check:

```bash
kubectl get pv
```

Initially:

```text
NAME     CAPACITY   ACCESS MODES   STATUS      CLAIM
my-pv    5Gi        RWO            Available
```

Important:

```text
Available
```

means:

> PV exists but nobody has claimed it yet.

---

# 6. Create PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 2Gi
```

Apply:

```bash
kubectl apply -f pvc.yaml
```

Check:

```bash
kubectl get pvc
```

You should see:

```text
NAME      STATUS   VOLUME   CAPACITY
my-pvc    Bound    my-pv    5Gi
```

And:

```bash
kubectl get pv
```

```text
NAME     CAPACITY   STATUS   CLAIM
my-pv    5Gi        Bound    default/my-pvc
```

So:

```text
PVC
 |
 | request: 2Gi
 v
PV
 |
 | capacity: 5Gi
 v
Actual storage
```

---

# 7. What exactly happened during binding?

This is important for interviews.

You created:

```text
PV
5Gi
RWO
```

Then:

```text
PVC
2Gi
RWO
```

Kubernetes looked for a matching PV.

It found:

```text
PV = 5Gi
PVC = 2Gi
```

Since:

```text
5Gi >= 2Gi
```

and access modes match:

```text
RWO == RWO
```

Kubernetes binds them:

```text
my-pvc  --->  my-pv
```

The PVC doesn't get a 2Gi slice of the PV.

The **PV remains 5Gi**.

---

# 8. Use PVC from a Pod

Now create:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: storage-test
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: app-storage
          mountPath: /usr/share/nginx/html

  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: my-pvc
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Architecture:

```text
                    Kubernetes
                        |
                        |
                      Pod
                  storage-test
                        |
                        | volumeMount
                        v
                /usr/share/nginx/html
                        |
                        v
                      PVC
                    my-pvc
                        |
                        v
                       PV
                     my-pv
                        |
                        v
                  /data/myapp
```

Now:

```bash
kubectl exec -it storage-test -- bash
```

Create data:

```bash
echo "Hello Kubernetes" > /usr/share/nginx/html/index.html
```

Check:

```bash
kubectl exec storage-test -- cat /usr/share/nginx/html/index.html
```

Output:

```text
Hello Kubernetes
```

---

# 9. What happens if Pod dies?

Delete the Pod:

```bash
kubectl delete pod storage-test
```

Create it again using the same PVC.

```bash
kubectl apply -f pod.yaml
```

Then:

```bash
kubectl exec storage-test -- cat /usr/share/nginx/html/index.html
```

You still get:

```text
Hello Kubernetes
```

Because:

```text
Pod
 ↓
PVC
 ↓
PV
 ↓
Persistent Storage
```

The Pod is disposable.

The storage isn't.

---

# 10. Why is manual PV management painful?

Imagine you have:

```text
100 applications
```

Each application needs:

```text
10Gi
```

You would need to manually create:

```text
PV-1
PV-2
PV-3
...
PV-100
```

And your storage administrator needs to provision the actual disks.

That's where **StorageClass** comes in.

---

# 11. StorageClass

A StorageClass tells Kubernetes:

> "When somebody requests storage through a PVC, use this storage backend and automatically create the required volume."

Instead of:

```text
Admin
 |
 | manually creates PV
 v
PV
 |
 v
PVC
```

you get:

```text
Developer
    |
    v
   PVC
    |
    v
StorageClass
    |
    v
Provisioner
    |
    v
Actual storage
```

The PV gets created automatically.

---

# 12. Manual vs Dynamic Provisioning

### Manual

```text
Administrator
     |
     v
Create PV
     |
     v
Developer creates PVC
     |
     v
PVC binds to PV
```

### Dynamic

```text
Developer
     |
     v
Create PVC
     |
     v
StorageClass
     |
     v
Provisioner
     |
     v
Create actual storage
     |
     v
Automatically create PV
     |
     v
PVC becomes Bound
```

This is the **big problem StorageClass solves**.

---

# 13. Real-world AWS example

Since you're working with EKS, this is the important architecture.

Suppose your application needs:

```text
20Gi EBS volume
```

Without dynamic provisioning:

```text
Admin
 |
 | manually creates EBS
 |
 v
PV
 |
 v
PVC
```

With StorageClass:

```text
Application
     |
     v
    PVC
  "20Gi RWO"
     |
     v
StorageClass
  "gp3"
     |
     v
EBS CSI Driver
     |
     v
AWS EBS
     |
     v
PV automatically created
```

So the developer only creates:

```yaml
PVC
```

and Kubernetes takes care of the rest.

---

# 14. Dynamic provisioning demo

Let's use a conceptual Kubernetes StorageClass.

For a real EKS cluster using the AWS EBS CSI driver, a StorageClass could look like:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com

parameters:
  type: gp3

reclaimPolicy: Delete

volumeBindingMode: WaitForFirstConsumer
```

Notice something important:

There is **no PV YAML**.

That's the whole point.

---

# 15. Create PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc
spec:
  accessModes:
    - ReadWriteOnce

  storageClassName: gp3

  resources:
    requests:
      storage: 20Gi
```

Apply:

```bash
kubectl apply -f pvc.yaml
```

Now Kubernetes sees:

```text
PVC
20Gi
 |
 | storageClassName: gp3
 v
StorageClass
 |
 v
EBS CSI Driver
 |
 v
AWS EBS 20Gi
```

The CSI driver provisions an EBS volume.

Then Kubernetes creates a PV representing that volume:

```text
PVC
 |
 v
PV
 |
 v
EBS Volume
```

---

# 16. Why `WaitForFirstConsumer`?

This is a **very important interview concept**.

Suppose you have:

```text
AZ-A
  Node A

AZ-B
  Node B
```

EBS volumes are AZ-specific.

If Kubernetes creates the EBS volume in AZ-A:

```text
EBS
AZ-A
```

but your Pod gets scheduled onto:

```text
Node B
AZ-B
```

you have a problem.

So with:

```yaml
volumeBindingMode: WaitForFirstConsumer
```

Kubernetes waits until it knows where the Pod will run.

```text
PVC created
     |
     v
Wait
     |
Pod created
     |
     v
Scheduler selects AZ-A
     |
     v
EBS created in AZ-A
     |
     v
PV created
     |
     v
PVC Bound
```

That's why `WaitForFirstConsumer` is commonly important for topology-aware storage.

---

# 17. Full dynamic architecture

```text
                 Developer
                     |
                     |
              creates PVC
                     |
                     v
              +-------------+
              |     PVC     |
              |    20 Gi    |
              |   gp3/RWO   |
              +-------------+
                     |
                     |
                     v
              +-------------+
              | StorageClass|
              |     gp3     |
              +-------------+
                     |
                     v
              +-------------+
              | CSI Driver  |
              | EBS CSI     |
              +-------------+
                     |
                     v
              +-------------+
              | AWS EBS     |
              |   20 Gi     |
              +-------------+
                     |
                     v
              +-------------+
              |     PV      |
              | automatically|
              |   created   |
              +-------------+
                     |
                     v
              +-------------+
              |     Pod     |
              +-------------+
```

---

# 18. Manual vs StorageClass — interview comparison

| Feature                   | Manual PV/PVC              | StorageClass |
| ------------------------- | -------------------------- | ------------ |
| PV creation               | Manual                     | Automatic    |
| PVC creation              | Manual                     | Manual       |
| Storage provisioning      | Manual                     | Automatic    |
| Storage admin involvement | High                       | Low          |
| Scalability               | Poor                       | Excellent    |
| Cloud environments        | Possible                   | Preferred    |
| Dynamic provisioning      | ❌                          | ✅            |
| Example                   | hostPath/NFS/static EBS PV | EBS CSI/gp3  |
| Developer experience      | More work                  | Simple       |

---

# 19. One important correction about terminology

People often say:

> "StorageClass creates the storage."

Technically, that's not quite right.

The flow is:

```text
PVC
 ↓
StorageClass
 ↓
Provisioner / CSI Driver
 ↓
Actual storage
```

For AWS:

```text
PVC
 ↓
StorageClass
 ↓
EBS CSI Driver
 ↓
AWS EBS
```

The **StorageClass defines the policy/configuration**.

The **CSI provisioner actually provisions the storage**.

---

# 20. Reclaim Policy — very important

You will commonly see:

```yaml
reclaimPolicy: Delete
```

or:

```yaml
reclaimPolicy: Retain
```

### Delete

```text
PVC deleted
   |
   v
PV deleted
   |
   v
Underlying storage deleted
```

Useful for temporary/application-managed data.

### Retain

```text
PVC deleted
   |
   v
PV retained
   |
   v
Underlying storage retained
```

Useful when you don't want accidental deletion of important data.

---

# 21. The complete mental model

Remember this:

```text
                  STORAGECLASS
                       |
                       | tells HOW
                       v
                    PROVISIONER
                       |
                       | creates
                       v
                  ACTUAL STORAGE
                       |
                       v
                       PV
                       |
                       | binds
                       v
                      PVC
                       |
                       | mounted by
                       v
                      POD
```

But in **manual provisioning**:

```text
Admin
 |
 | creates
 v
PV
 |
 | binds
 v
PVC
 |
 | mounted by
 v
Pod
```

And in **dynamic provisioning**:

```text
Developer
 |
 | creates
 v
PVC
 |
 | references
 v
StorageClass
 |
 v
CSI Provisioner
 |
 v
Actual Storage
 |
 v
PV automatically created
 |
 v
PVC Bound
 |
 v
Pod
```

## Reclaiming

When a user is done with their volume, they can delete the PVC objects from the API that allows reclamation of the resource. The reclaim policy for a PersistentVolume tells the cluster what to do with the volume after it has been released of its claim. Currently, volumes can either be Retained, Recycled, or Deleted.

## Retain

The Retain reclaim policy allows for manual reclamation of the resource. When the PersistentVolumeClaim is deleted, the PersistentVolume still exists and the volume is considered "released". But it is not yet available for another claim because the previous claimant's data remains on the volume. An administrator can manually reclaim the volume with the following steps.

    Delete the PersistentVolume. The associated storage asset in external infrastructure still exists after the PV is deleted.
    Manually clean up the data on the associated storage asset accordingly.
    Manually delete the associated storage asset.

If you want to reuse the same storage asset, create a new PersistentVolume with the same storage asset definition.
## Delete

For volume plugins that support the Delete reclaim policy, deletion removes both the PersistentVolume object from Kubernetes, as well as the associated storage asset in the external infrastructure. Volumes that were dynamically provisioned inherit the reclaim policy of their StorageClass, which defaults to Delete. The administrator should configure the StorageClass according to users' expectations; otherwise, the PV must be edited or patched after it is created. See Change the Reclaim Policy of a PersistentVolume.

## Recycle
>Warning:
- The Recycle reclaim policy is deprecated. Instead, the recommended approach is to use dynamic provisioning.

If supported by the underlying volume plugin, the Recycle reclaim policy performs a basic scrub (rm -rf /thevolume/*) on the volume and makes it available again for a new claim.

### Interview answer in one sentence

> **PV is cluster storage, PVC is a user's request for storage, and StorageClass enables dynamic provisioning so that Kubernetes can automatically provision the underlying storage and create the corresponding PV instead of an administrator manually creating PVs.**

For your **EKS interview**, the next thing I'd focus on is **EBS CSI driver → StorageClass → PVC → PV → EBS**, including **`WaitForFirstConsumer`, AZ constraints, `RWO`, snapshots, expansion, and StatefulSets**.
