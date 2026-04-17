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
