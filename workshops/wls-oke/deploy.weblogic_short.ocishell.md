# Deploy a WebLogic domain  ###


## Before You Begin
### Objectives
- Prepare the Kubernetes cluster to run WebLogic domains.
- Update the Traefik load balancer and operator configuration.
- Deploy a WebLogic domain on Kubernetes.

### Introduction

Thus far on our journey from On-premises WebLogic Server to Oracle Container Engine for Kubernetes, we have created Oracle Container Engine for Kubernetes (OKE) on Oracle Cloud Infrastructure (OCI), installed and configured the WebLogic Kubernetes operator, installed and configured the Traefik. Now we need to deploy a WebLogic domain in OKE using WebKogic Kubernetes Operator.

During this lab, you will deploy a WebLogic domain in OKE using WebKogic Kubernetes Operator.

## Required Artifacts

- You should already have completed labs 1, 2, and 3 before beginning this lab.

- **Works better with the Chrome browser**.

## Task 1: Prepare the Kubernetes cluster to run WebLogic domains  
Create the domain namespace:
```bash
<copy>kubectl create namespace sample-domain1-ns</copy>
```
Create a Kubernetes secret containing the Administration Server boot credentials:
```bash
<copy>kubectl -n sample-domain1-ns create secret generic sample-domain1-weblogic-credentials \
  --from-literal=username=weblogic \
  --from-literal=password=welcome1</copy>
```

## Task 2: Update the Traefik load balancer and operator configuration ####

After you have your domain namespace (the WebLogic domain is not deployed yet), you have to update the load balancer and operator configuration to specify where the domain will be deployed.

Before executing the domain **helm** install, be sure that you are in the WebLogic operator local Git repository folder.

```
<copy>cd ~/weblogic-kubernetes-operator/</copy>
```
To update the operator, execute the following **helm upgrade** command:
```
<copy>helm upgrade sample-weblogic-operator \
  kubernetes/charts/weblogic-operator \
  --namespace sample-weblogic-operator-ns \
  --reuse-values \
  --set "domainNamespaces={sample-domain1-ns}" \
  --wait</copy>
```

To update Traefik, execute the following **helm upgrade** command:
```bash
<copy>helm upgrade traefik-operator \
  traefik/traefik \
  --namespace traefik \
  --reuse-values \
  --set "kubernetes.namespaces={traefik,sample-domain1-ns}" \
  --wait</copy>
```
Note that in both cases, the only updated parameter is the domain namespace.

## Task 3: Deploy a WebLogic domain on Kubernetes ####

To deploy WebLogic domain, you need to create a domain resource definition which contains the necessary parameters for the operator to start the WebLogic domain properly.

We provided for you a **domain.yaml** file that contains a YAML representation of the custom resource object. Please copy it locally:
```bash
<copy>curl -LSs https://raw.githubusercontent.com/nagypeter/weblogic-operator-tutorial/master/k8s/domain_short.yaml >~/domain.yaml</copy>
```
Review it in your favorite editor or a [browser](../domain.yaml).

Create the domain custom resource object with the following command:
```bash
<copy>kubectl apply -f ~/domain.yaml</copy>
```
Check the introspector job, which needs to be run first:
```bash
<copy>kubectl get pod -n sample-domain1-ns</copy>
```
```bash
NAME                                         READY     STATUS              RESTARTS   AGE
sample-domain1-introspect-domain-job-kcn4n   0/1       ContainerCreating   0          7s
```
Periodically check the pods in the domain namespace and soon you will see the servers starting:
```bash
<copy>kubectl get po -n sample-domain1-ns -o wide</copy>
```
```bash
NAME                             READY     STATUS    RESTARTS   AGE       IP            NODE            NOMINATED NODE
sample-domain1-admin-server      1/1       Running   0          2m        10.244.2.10   130.61.84.41    <none>
sample-domain1-managed-server1   1/1       Running   0          1m        10.244.2.11   130.61.84.41    <none>
sample-domain1-managed-server2   0/1       Running   0          1m        10.244.1.4    130.61.52.240   <none>
```
You should see three running pods similar to the results shown above. If you don't see all the running pods, wait and then check periodically. The entire domain deployment may take up to 2-3 minutes depending on the compute shapes.

In order to access any application or the Administration Console deployed on WebLogic, you have to configure a **Traefik** Ingress. An OCI load balancer is already assigned during the **Traefik** install in the previous step.

As a simple solution, it's best to configure path routing, which will route external traffic through **Traefik** to the domain cluster address or the Administration Server Console.

Execute the following Ingress resource definition:
```bash
<copy>cat << EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-pathrouting-1
  namespace: sample-domain1-ns
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host:
    http:
      paths:
      - path: /
        backend:
          serviceName: sample-domain1-cluster-cluster-1
          servicePort: 8001
      - path: /console
        backend:
          serviceName: sample-domain1-admin-server
          servicePort: 7001          
EOF</copy>
```


Please note the two backends and the namespace, **serviceName**, **servicePort** definitions. The first backend is the domain cluster service to reach the application at the root context path. The second is for the admin console which is a different service.

Once the Ingress has been created construct the URL of the Administration Console based on the following pattern:

`http://EXTERNAL-IP/console`

The `EXTERNAL-IP` was determined during the Traefik install. If you forgot to note it, then execute the following command to get the public IP address:
```bash
<copy>kubectl describe svc traefik-operator --namespace traefik | grep Ingress | awk '{print $3}'</copy>
```
```bash
129.213.172.44
```
Construct the Administration Console URL and open it in a browser:

Enter the administrative user credentials (weblogic/welcome1) and click **Login**.

![](images/deploy.domain/weblogic.console.login.png)

!Please note in this use case that the use of the Administration Console is just for demo/test purposes because the domain configuration is persisted in the pod, which means that after the restart, the original values (baked into the image) will be used again. To override certain configuration parameters - to ensure image portability - follow the override part of this tutorial.

#### Test the sample web application ####

The URL pattern of the sample application is the following:

`http://EXTERNAL-IP/opdemo/?dsname=testDatasource`

![](images/deploy.domain/webapp.png)

Refresh the page and notice the hostname changes. It reflects the Managed Server's name which responds to the request. You should see load balancing between the two Managed Servers.
## Acknowledgements

- **Authors/Contributors** - 
- **Last Updated By/Date** - 
- **Workshop Expiration Date** - April 31, 2021

## Need Help?
Please submit feedback or ask for help using our [LiveLabs Support Forum](https://community.oracle.com/tech/developers/categories/livelabsdiscussions). Please click the **Log In** button and login using your Oracle Account. Click the **Ask A Question** button to the left to start a *New Discussion* or *Ask a Question*.  Please include your workshop name and lab name.  You can also include screenshots and attach files.  Engage directly with the author of the workshop.

If you do not have an Oracle Account, click [here](https://profile.oracle.com/myprofile/account/create-account.jspx) to create one.    Please include the workshop name and lab in your request.
