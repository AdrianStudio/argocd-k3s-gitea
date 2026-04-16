# argocd-k3s-gitea
Guide to deploy ArgoCD on K3S and connect it with Gitea

My K3S cluster is built with 3 Raspberry Pis, a Pi-master, and 2 Pi-workers. Everything will be installed from the master downwards.

First step is to create the **namespace**, a **namespace** is like a logical folder that separates workloads. ArgoCD will go in its own **namespace** _argocd_ to keep it isolated. You can use any name you want
To create the namespace, we will use the command **_sudo kubectl create namespace argocd_** and then confirm its created with **_sudo kubectl get namespace_**.

<img width="504" height="130" alt="image" src="https://github.com/user-attachments/assets/57d7cb0e-10b8-4dd3-a5df-3660214de0d0" />
