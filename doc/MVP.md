# Minimum Viable Product (MVP):

`Goal`: Implement and lauch basix version of a product that includes only essential features in order to receive early feedback and validate core idea and concept,
reduce time to market, minimize consts, mitigate risks associated with building a full-featured product without sufficient validation.

## [Link to demo video](https://drive.google.com/file/d/1bsbCrDLBlJqXVR_-t9qdhhWslv8TtA8w/view?usp=sharing)

1. Check functionality of the application
- Forward the ports with the following command:
```bash
$ k port-forward -n demo svc/ambassador 8081:80&
Forwarding from 127.0.0.1:8081 -> 80
Forwarding from [::1]:8081 -> 80
```
- Make request to this port and check an answer as the app version:
```bash
$ curl localhost:8081
k8sdiy-api:599e1af#
```

2. Fix config issue
- Check network config of the app:
```bash
$ k get svc -n demo
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)               AGE
ambassador         LoadBalancer   10.43.190.212   <pending>     80:30092/TCP          84s
```
- In [repo with helm config files](https://github.com/vit-um/go-demo-app/blob/master/helm/values.yaml) change `api-gateway:` to `NodePort`
- Sync and apply these changes using ArgoCD
- Check the result
```bash
✗ k get svc -n demo
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
ambassador         NodePort       10.43.190.212   <none>        80:30092/TCP    84s
```

3. Check the app behavior after fixing config issue.
- Upload file from local repo to remote server:
```bash
curl -F 'image=@g.png' localhost:8081/img/
```
- See result in console
