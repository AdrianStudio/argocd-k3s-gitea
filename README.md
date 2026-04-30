# argocd-k3s-gitea
Guide to deploy ArgoCD on K3s and connect it with Gitea

My K3s cluster is built with 3 Raspberry Pis, a Pi-master, and 2 Pi-workers. Everything will be installed from the master downwards.

**Why GitOps? The decision behind this setup**

Managing a K3s cluster manually, applying manifests with kubectl, debugging errors through terminal output, tracking what's deployed where, gets complex fast. Every change requires direct cluster access, and there's no single source of truth for what's actually running.
I implemented GitOps with ArgoCD and Gitea to solve this. Instead of running commands directly against the cluster, I define the desired state in a YAML file, push it to Gitea, and ArgoCD takes care of the rest.

The real advantages this gives me day to day:

**_Visual deployment feedback:_** ArgoCD shows exactly what's happening during a sync, surfaces errors clearly, and confirms the health of every resource in the namespace

**_No direct cluster management:_** I don't create or delete namespaces, deployments, or services manually. Git is the only interface

**_Production-like workflow:_** the same GitOps principles used in real engineering teams at scale, running on Raspberry Pis

This turns a homelab into something that operates like a real platform, where Git is the source of truth and the cluster reflects exactly what's in the repository at all times.

---

**Step 1 - Create ArgoCD's namespace**

The first step is to create the namespace. A namespace is like a logical folder that separates workloads. ArgoCD will go in its own namespace `argocd` to keep it isolated from other services. You can use any name you want.

To create the namespace, run `sudo kubectl create namespace argocd`, then confirm it was created with `sudo kubectl get namespaces`.

<img width="504" height="130" alt="image" src="https://github.com/user-attachments/assets/57d7cb0e-10b8-4dd3-a5df-3660214de0d0" />

---

**Step 2 - Install ArgoCD**

K3s already has **Helm** available. **Helm** is a Kubernetes package manager; the same way `apt` installs Debian packages, **Helm** installs applications in Kubernetes. A **Helm** "chart" is the package that contains all the necessary YAML manifests to deploy an app.

To install ArgoCD, we will use the official manifest; it's the most direct and recommended way. Run `sudo kubectl apply -n argocd -f` followed by the official URL.

<img width="1183" height="91" alt="image" src="https://github.com/user-attachments/assets/fc19b9e5-4641-4954-9fd7-c4b57cc7837f" />

This will download the official ArgoCD manifest and create all necessary resources in the namespace `argocd`, such as deployments, services, configmaps, RBAC, etc.

Once the installation is done, check the available pods to confirm everything is running properly with `sudo kubectl get pods -n argocd`.

<img width="815" height="169" alt="image" src="https://github.com/user-attachments/assets/74d18abb-37d9-47e9-b26d-6f577d439eac" />

ArgoCD has a web UI. By default, the service `argocd-server` is of type _ClusterIP_ — only accessible from inside the cluster. To access it from a browser, we need to expose it with a **NodePort**, which assigns a fixed port to all nodes in the cluster.

---

**Step 3 - Expose ArgoCD with NodePort**

The following command changes the service type from `ClusterIP` to `NodePort` and assigns port `30443`. Verify with `sudo kubectl get svc argocd-server -n argocd`.

<img width="1433" height="111" alt="image" src="https://github.com/user-attachments/assets/3537e4aa-0e63-44ca-a538-9e41456c0e1d" />

After running these commands, access ArgoCD from your browser using the node IP and the assigned port. The credentials are:

- **Username:** `admin`
- **Password:** retrieve it with the following command:

<img width="1121" height="41" alt="image" src="https://github.com/user-attachments/assets/baf9c613-3eba-4c5a-b159-7529f4136b6b" />

---

**Step 4 - Change ArgoCD's password**

The initial password is temporary and must be changed. In the left sidebar, go to _User Info_ and click the button to update your password.

<img width="558" height="169" alt="image" src="https://github.com/user-attachments/assets/82ba29ee-5401-4b4a-b6f3-1f4ef617c808" />

---

**Step 5 - Create a folder in Gitea for the manifests**

If you don't have a repository in Gitea, create one. Once that's done, create a new folder inside it — this is where all the manifests that ArgoCD manages will go. You can choose any name and location.

In Gitea, you can't create an empty folder directly. Click _Add file_ and write the path as `argocd/.gitkeep` — Gitea will create the `argocd` folder automatically. The `.gitkeep` file is an empty file used by convention to keep empty folders tracked in Git.

<img width="1105" height="271" alt="image" src="https://github.com/user-attachments/assets/87d25b80-ca51-4497-bccc-e68f741b594c" />

---

**Step 6 - Connect Gitea's repository to ArgoCD**

This tells ArgoCD where your repository is and how to access it. From this moment, ArgoCD will read the YAML files you add to the repository and apply them automatically to the cluster once it detects changes.

> **This is how it works:** You edit the YAML on Gitea → ArgoCD detects the change → it applies it to the cluster automatically. Without this step, ArgoCD won't know where to look. This is the bridge between Git and Kubernetes.

Go to _Settings_ on the left sidebar → _Repositories_ → _Connect Repo_.

<img width="1464" height="576" alt="image" src="https://github.com/user-attachments/assets/59a1f6f5-6c0a-47f2-a164-0df861f9475f" />

Select _VIA HTTP/HTTPS_, give it a name, set project to `default`, enter your repository URL, and your Gitea username and password. Click **Connect**.

After a few seconds you should see the repository listed with a green **Successful** status.

<img width="1261" height="187" alt="image" src="https://github.com/user-attachments/assets/3764dc1c-32be-479c-9dca-bc99756de43f" />

---

**Step 7 - Create a test manifest in Gitea**

A **manifest** is a YAML file that describes to Kubernetes what you want to deploy. In this case, we will deploy a test nginx server. ArgoCD will read this file from Gitea and apply it to the cluster automatically.

Create the file at `k3s/argocd/nginx-test.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-test
  namespace: homelab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-test
  template:
    metadata:
      labels:
        app: nginx-test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
```

In Gitea, go to the repository → open the folder → click _Add file_ → paste the content → commit.

<img width="793" height="217" alt="image" src="https://github.com/user-attachments/assets/42eff275-61cc-4aa3-82c7-2f30fb9fcca2" />

<img width="926" height="564" alt="image" src="https://github.com/user-attachments/assets/afe7d0c0-1eb9-4620-9e4d-3c2e3e424a01" />

---

**Step 8 - Create the Application in ArgoCD**

An **Application** in ArgoCD is the object that connects everything. It tells ArgoCD: _"monitor this folder in Gitea and make sure the cluster always matches exactly what's there."_

With _Automatic Sync_ enabled, every time you push a commit to Gitea, ArgoCD detects the change and applies it to the cluster without any manual intervention. With _Self Heal_ enabled, if anyone deletes a pod manually, ArgoCD restores it automatically from Git.

Go to **ArgoCD UI → Applications → New App** and fill it in as follows:

<img width="810" height="456" alt="image" src="https://github.com/user-attachments/assets/fcdcc2dc-f96f-46a5-8136-1a97e077be06" />

<img width="786" height="400" alt="image" src="https://github.com/user-attachments/assets/07498d4b-254f-4754-abb5-cd89c8ec5c35" />

<img width="789" height="332" alt="image" src="https://github.com/user-attachments/assets/14b0e931-4d88-45f2-a00f-1e28f0b0b05c" />

Click **Create**. You should see the application listed with **Healthy + Synced** status. To confirm the test nginx was deployed, run `sudo kubectl get pods -n homelab` — you should see `nginx-test` alongside your other services.

<img width="788" height="188" alt="image" src="https://github.com/user-attachments/assets/fe803a49-d9d3-44ea-b4cb-680aeda08710" />

---

**Step 9 - Verify the complete GitOps pipeline**

Now we'll prove the pipeline works end to end. Make a change in Gitea and watch ArgoCD apply it automatically — no touching the cluster directly.

Go to **Gitea → repository → edit `nginx-test.yaml` → change `replicas: 1` to `replicas: 2` → commit**.

Wait 30–60 seconds and run `sudo kubectl get pods -n homelab`. You should see 2 `nginx-test` pods instead of 1. This confirms the full flow works:

> **Git push → ArgoCD detects the change → applies to the cluster automatically**

<img width="780" height="215" alt="image" src="https://github.com/user-attachments/assets/e679a9b9-c985-48e2-b923-d7112ebfb305" />

After verifying, delete the `nginx-test.yaml` file from Gitea. With **Prune Resources** enabled, ArgoCD will automatically remove the deployment from the cluster as well.

From this point, your Git repository is the single source of truth for your cluster:

- Want to add a service? Create the YAML in Gitea.
- Want to scale a deployment? Edit the YAML in Gitea.
- Want to remove a service? Delete the file from Gitea.

**The real advantages this provides:**

- **Full history** — every change made to the cluster is recorded in Git with date, author, and commit message.
- **Instant rollback** — if anything breaks, run `git revert` and ArgoCD restores the previous state.
- **Reproducibility** — if a node dies, you can rebuild the entire cluster state from Git in minutes.

---

## Found this useful?

If this guide helped you set up ArgoCD on K3s with Gitea, feel free to ⭐ the repo. If you run into issues or have suggestions, open an issue — happy to help. This is part of a larger homelab project where I document everything I build and learn, so contributions and feedback are always welcome.
