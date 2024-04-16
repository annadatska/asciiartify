## ArgoCD. Proof of Concept

`ArgoCD` is a Kubernetes controller that continuously monitors running applications and compares the current state with the desired state.
A deployment whose current state differs from the target is considered `out of sync'.
ArgoCD informs and visualizes the differences, providing opportunities for automatic or manual synchronization of the desired state.

`Goal`: To prove the viability of the idea and concept of using ArgpCD as a CD tool. We'll prove that it's technically possible to implement the idea.
[Link to step-by-step demo video](https://drive.google.com/file/d/13mkXIxbJBYPvNcWQBGJgv2Fxh6W_0BXl/view?usp=sharing)
1. Prepare and set up a separate local cluster for ArgoCD installation:
```bash
$ k3d cluster create argo
... 
INFO[0029] Cluster 'argo' created successfully!         
INFO[0029] You can now use it like this: kubectl cluster-info

$ kubectl cluster-info
Kubernetes control plane is running at https://0.0.0.0:43297
CoreDNS is running at https://0.0.0.0:43297/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:43297/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

$ k version
$ k get all -A
```

2. Installation
   We have the option to install ArgoCD using Helm, but for this setup, we'll utilize a script from the official ArgoCD repository [ArgoCD](https://argo-cd.readthedocs.io/en/stable/#quick-start).
   Initially, we'll create a dedicated namespace for the system installation, followed by executing the installation using the script (manifest)
   and then verifying the system's state post-installation.
```bash
$ kubectl create namespace argocd
namespace/argocd created

$ k get ns
NAME              STATUS   AGE
kube-system       Active   80m
default           Active   80m
kube-public       Active   80m
kube-node-lease   Active   80m
argocd            Active   28s

$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

$ k get all -n argocd

# check containers status: 
$ k get pod -n argocd -w
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-redis-b5d6bf5f5-sqlgw                        1/1     Running   0          3m41s
argocd-notifications-controller-589b479947-fm4kp    1/1     Running   0          3m41s
argocd-applicationset-controller-6f9d6cfd58-9rvjf   1/1     Running   0          3m41s
argocd-dex-server-6df5d4f8c4-nd829                  1/1     Running   0          3m41s
argocd-application-controller-0                     1/1     Running   0          3m40s
argocd-server-7459448d56-xnrvf                      1/1     Running   0          3m40s
argocd-repo-server-7b75b45897-qsw9z                 1/1     Running   0          3m41s
```

3. Get access to ArgoCD GUI:
Use `Port Forwarding` with local port `8080`. n the command, we refer to the `svc/argocd-server` service located in the `-n argocd` namespace.
Kubectl will automatically find the service endpoint and set port forwarding from local port 8080 to remote port 443
```bash
$ kubectl port-forward svc/argocd-server -n argocd 8080:443&
[1] 23820
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
```
ArgoCD works with https by default, so when we try to open [127.0.0.1:8080](https://127.0.0.1:8080/) we get an error certificates.
Therefore, in a productive system, you need to install certificates and configure these points.

4. Get credentials
Use the command to get the password, specify the secret file `argocd-initial-admin-secret` and the output format `jsonpath="{.data.password}"`.
It will return base64 encoded password. Then use the `base64 -d` command to return the password to plain text.
Enter received password and `admin' login into the ArgoCD Web interface.
```bash
$ k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
cFZIWTM3eFpJR3hDeUxLNQ==#                                                                                                        
$ k -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"|base64 -d;echo
pVHY37xZIGxCyLK5
```

5. Create new app usinf GUI
- Click `+ NEW APP` 
- Enter Application name `demo`
- Project name `default`
- Sync policy leave as is (`Manual`) and check box `AUTO-CREATE NAMESPACE`
- In `SOURCE` tab leave type as is - `GIT`
- Enter `url` repo with manifests for deployment https://github.com/annadatska/go-demo-app
- Select path to catalog `helm` in `Path` field
- In `DESTINATION` tab put `url` of local cluster and `Namespace` demo.
- У розділі політика синхронізація вкажемо як додаток буде синхронізуватись з репозиторієм. Тут важливо вказати ArgoCD щоб створив новий namespace так як в helm цю функцію за замовчуванням прибрали. Ставимо галку напроти `AUTO-CREATE NAMESPACE`   
- Створюємо додаток кнопкою `CREATE`

6. Review the details of the deployed application by clicking on it in the list.
GUI provides a hierarchical view of the application components, their deployment and current state in the cluster.

7. App synchronization
- Click `SYNC` and then `SYNCHRONIZE` button
- Check app status in cluster

8. Check ArgoCD behavior after the changes in the repo.
- Change gateway type from `NodePort` to `LoadBalancer` in https://github.com/vit-um/go-demo-app/blob/master/helm/values.yaml
```bath
$ k get svc -n demo
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                 AGE
demo-nats          ClusterIP   None            <none>        4222/TCP,6222/TCP,8222/TCP,7777/TCP,7422/TCP,7522/TCP   31m
demo-front         ClusterIP   10.43.247.92    <none>        80/TCP                                                  31m
cache              ClusterIP   10.43.234.48    <none>        6379/TCP                                                31m
ambassador         NodePort    10.43.190.212   <none>        80:30092/TCP                                            31m
```
- Sync call will lead to updated git repo version and compare it with  the current state
```bath
$ NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
ambassador         LoadBalancer   10.43.190.212   <pending>     80:30092/TCP          35m
```
