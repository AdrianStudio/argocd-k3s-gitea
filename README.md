# argocd-k3s-gitea
Guide to deploy ArgoCD on K3S and connect it with Gitea

My K3S cluster is built with 3 Raspberry Pis, a Pi-master, and 2 Pi-workers. Everything will be installed from the master downwards.

**Step 1 - Create ArgoCD's namespace**

First step is to create the **namespace**, a **namespace** is like a logical folder that separates workloads. ArgoCD will go in its own **namespace** _argocd_ to keep it isolated. You can use any name you want
To create the namespace, we will use the command **_sudo kubectl create namespace argocd_** and then confirm its created with **_sudo kubectl get namespace_**.

<img width="504" height="130" alt="image" src="https://github.com/user-attachments/assets/57d7cb0e-10b8-4dd3-a5df-3660214de0d0" />

The next step is to use **Helm**, which is a Kubernetes package manager. The same way **_apt_** installs **Debian** packages, **Helm** installs apps in Kubernetes. A Helm "chart" is the package that contains all necessary YAML manifests to deploy an app.
**ArgoCD** has its own official chart that we will use.

**Sept 2 - Install ArgoCD**

k3S already has **Helm** available. To install ArgoCD, we will use the official manifest; it's the best and most direct way to install it. We will use the command **_sudo kubectl apply -n argocd -f (and the official URL)_**

<img width="1183" height="91" alt="image" src="https://github.com/user-attachments/assets/fc19b9e5-4641-4954-9fd7-c4b57cc7837f" />

This will download the official ArgoCD manifest, and it creates all necessary resources in the namespace _argocd_, such as deployments, services, configmaps, RBAC, etc.
Once the installation is done, we will check the available pods to confirm everything is running properly with the command **_sudo kubectl get pods -n argocd_**.

<img width="815" height="169" alt="image" src="https://github.com/user-attachments/assets/74d18abb-37d9-47e9-b26d-6f577d439eac" />

ArgoCD has a UI web. By default, the service _argocd-server_ is of type _ClusterIP_; only accessible from inside the cluster. To access from our web browser, we will need to expose it. We will do this with a **NodePort**, which will assign a fixed port to all the nodes in the cluster, to access from our local network.

**Step 3 - Expose ArgoCD with NodePort**

With the next command, we will change the type of service, from ClusterIP to NodePort and assign the port that we need, in this case 30443. And we will verify with **_sudo kubectl get svc argocd-server -n argocd_**

<img width="1433" height="111" alt="image" src="https://github.com/user-attachments/assets/3537e4aa-0e63-44ca-a538-9e41456c0e1d" />

After running these commands and verifying that they applied, we will access ArgoCD from our web browser, entering the Cluster's/Node IP and the port we assigned. 
The credentials are:
  username: **admin**
  password: We will obtain it with a command
  <img width="1121" height="41" alt="image" src="https://github.com/user-attachments/assets/baf9c613-3eba-4c5a-b159-7529f4136b6b" />

**Step 4 - Change ArgoCD's password**

The initial password is temporary, and it has to be changed. On the start menu on the left, go to _User Info_, and you will see a button to update your password. The next step will be connecting Gitea.

<img width="558" height="169" alt="image" src="https://github.com/user-attachments/assets/82ba29ee-5401-4b4a-b6f3-1f4ef617c808" />

**Step 5 - Connect Gitea to ArgoCD**

If you don't have a repository inside Gitea, you will have to create one. Once that's done, you will create a new folder inside. You can choose any name; I will create mine inside another folder where I manage all my K3S. This is where all the manifests that ArgoCD manages will go.

In Gitea, you can't create an empty folder; you will have to click new file > and write it down like this: argocd/.gitkeep, so Gitea creates the folder _argocd_ and the file .gitkeep is an empty file used to keep empty folders active in Git.

<img width="1105" height="271" alt="image" src="https://github.com/user-attachments/assets/87d25b80-ca51-4497-bccc-e68f741b594c" />


**Step 6 - Connect Gitea's Repository to ArgoCD**

This will tell ArgoCD where our repository is and how to access it; from this moment, ArgoCD will read the YAML files that you add to the repository. Once it detects changes to the folder, it will apply them automatically to the cluster.

**This is how it works: You edit the YAML on Gitea > ArgoCD detects the change > it applies it to the cluster automatically.** Without this, ArgoCD won't know where to look. This is the bridge between Git and Kubernetes.

To connect the repository to ArgoCD go to _Settings_ on the left side menu > _Repositories_ > _Connect Repo_

<img width="1464" height="576" alt="image" src="https://github.com/user-attachments/assets/59a1f6f5-6c0a-47f2-a164-0df861f9475f" />



Select via _HTTP/HTTPS_, name of your choice, project: default, your repository URL, and the username and password that you have set up on Gitea. And then click **Connect**
Once you click connect, you should see the repository after a few seconds, with a Succesful green tick, meaning its connected.

<img width="1261" height="187" alt="image" src="https://github.com/user-attachments/assets/3764dc1c-32be-479c-9dca-bc99756de43f" />


**Step 7 - Create a test Manifest on Gitea**

A manifest is a _YAML_ file that describes to Kubernetes what you want to deploy. In this case, we will deploy an nginx test server. ArgoCD will read this file from Gitea, and it will apply it to the cluster automatically.

Create the file on the route you created the file, in my case _k3s/argocd/nginx-test.yaml_ with this content:

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

To create it, go to the repository you created, then into the folder, click add file, paste the content and commit.

<img width="793" height="217" alt="image" src="https://github.com/user-attachments/assets/42eff275-61cc-4aa3-82c7-2f30fb9fcca2" />

<img width="926" height="564" alt="image" src="https://github.com/user-attachments/assets/afe7d0c0-1eb9-4620-9e4d-3c2e3e424a01" />


**Step 8 - Create the app on ArgoCD**

An app on ArgoCD is the object that connects everything; it tells ArgoCD, "check this folder on Gitea, make sure that the cluster replicates exactly what's on there"

With _Automatic Sync_ activated, every time we do a commit on Gitea, ArgoCD will detect this, and it will apply the changes on the cluster, without having to edit anything. With _Self Heal_, if anyone deletes a pod manually, ArgoCD will restore it automatically from Git.

Go to **ArgoCD UI > Applications > New App** and fill it like this:

<img width="810" height="456" alt="image" src="https://github.com/user-attachments/assets/fcdcc2dc-f96f-46a5-8136-1a97e077be06" />

<img width="786" height="400" alt="image" src="https://github.com/user-attachments/assets/07498d4b-254f-4754-abb5-cd89c8ec5c35" />

<img width="789" height="332" alt="image" src="https://github.com/user-attachments/assets/14b0e931-4d88-45f2-a00f-1e28f0b0b05c" />







