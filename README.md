
 <img src="https://github.com/afreisinger/jenkins/blob/main/img/logo-title-opengraph.png?raw=true" width="398" />

## Install Jenkins with YAML files on Kind cluster running on Mac

## Prerequisites

### Docker
Refer to the [Docker Engine installation instructions](https://docs.docker.com/engine/install/) for your platform.

### Kubernetes
If you don’t have a running Kubernetes cluster, see the [Create a Kind cluster](https://kind.sigs.k8s.io/) section.

For more detailed instructions see [the user guide](https://kind.sigs.k8s.io/docs/user/quick-start).

Install kubectl. See the instructions from the [Install and Set Up kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) page.

## Usage

### Create a Kind multinode cluster

```
$ kind create cluster --config 00-kind-cluster-multinode.yaml --name=k8s
```
* `- download file (YAML)`, [00-kind-cluster-multinode.yaml](https://raw.githubusercontent.com/afreisinger/jenkins/main/00-kind-cluster-multinode.yaml)
 

### [Create a namespace](#create-a-namespace)
A distinct namespace provides an additional layer of isolation and more control over the continuous integration environment. Create a namespace for the Jenkins deployment by typing the following command on your terminal:

```
$ kubectl create namespace jenkins
```
Use the following command to list existing namespaces:

```
$ kubectl get namespaces
```

The output confirms that the jenkins namespace was created successfully.

```
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   50m
jenkins           Active   30s
kube-node-lease   Active   50m
kube-public       Active   50m
kube-system       Active   50m
```

Use the following command to set a namespace by default

```
kubectl config set-context --current --namespace=jenkins
```


### Install Jenkins with YAML files
This section describes how to use a set of YAML (Yet Another Markup Language) files to install Jenkins on a Kubernetes cluster. The YAML files are easily tracked, edited, and can be reused indefinitely.

### Create Jenkins deployment file
Copy the contents [here](https://raw.githubusercontent.com/afreisinger/jenkins/main/01-jenkins-deployment.yaml) into your preferred text editor and create a 01-jenkins-deployment.yaml file in the “jenkins” namespace we created in this [section](#create-a-namespace) above.

* This [deployment file](https://raw.githubusercontent.com/afreisinger/jenkins/main/01-jenkins-deployment.yaml) is defining a Deployment as indicated by the kind field.
* The Deployment specifies a single replica. This ensures one and only one instance will be maintained by the Replication Controller in the event of failure.

* The container image name is jenkins and version is 2.32.2

* The list of ports specified within the spec are a list of ports to expose from the container on the Pods IP address.

    * Jenkins running on (http) port 8080.

    * The Pod exposes the port 8080 of the jenkins container.

* The volumeMounts section of the file creates a Persistent Volume. This volume is mounted within the container at the path /var/jenkins_home and so modifications to data within /var/jenkins_home are written to the volume. This volume is mounted within the container at the path /var/jenkins_home and so modifications to data within /var/jenkins_home are written to the volume. The role of a persistent volume is to store basic Jenkins data and preserve it beyond the lifetime of a pod.

Exit and save the changes once you add the content to the Jenkins deployment file.

### Deploy Jenkins
To create the deployment execute:
```
$ kubectl create -f 01-jenkins-deployment.yaml
```
* `- download file (YAML)`, [01-jenkins-deployment.yaml](https://raw.githubusercontent.com/afreisinger/jenkins/main/01-jenkins-deployment.yaml)
 


The command also instructs the system to install Jenkins within the jenkins namespace.

To validate that creating the deployment was successful you can invoke:

```
$ kubectl get deployments
```

### Grant access to Jenkins service

We have a Jenkins instance deployed but it is still not accessible. The Jenkins Pod has been assigned an IP address that is internal to the Kubernetes cluster. It’s possible to log into the Kubernetes Node and access Jenkins from there but that’s not a very useful way to access the service.

To make Jenkins accessible outside the Kubernetes cluster the Pod needs to be exposed as a Service. A Service is an abstraction that exposes Jenkins to the wider network. It allows us to maintain a persistent connection to the pod regardless of the changes in the cluster. With a local deployment, this means creating a NodePort service type. A NodePort service type exposes a service on a port on each node in the cluster. The service is accessed through the Node IP address and the service nodePort. A simple service is defined [here](https://raw.githubusercontent.com/afreisinger/jenkins/main/02-jenkins-service.yaml):

* This [service file](https://raw.githubusercontent.com/afreisinger/jenkins/main/02-jenkins-service.yaml) is defining a Service as indicated by the `kind` field.

* The Service is of type NodePort. Other options are ClusterIP (only accessible within the cluster) and LoadBalancer (IP address assigned by a cloud provider e.g. AWS Elastic IP).

* The list of ports specified within the spec is a list of ports exposed by this service.

    * The port is the port that will be exposed by the service.

    * The target port is the port to access the Pods targeted by this service. A port name may also be specified.

The selector specifies the selection criteria for the Pods targeted by this service.

To create the service execute:

```
$ kubectl create -f 02-jenkins-service.yaml
```
  * `- download file (YAML)`, [02-jenkins-service.yaml](https://raw.githubusercontent.com/afreisinger/jenkins/main/02-jenkins-service.yaml)

To validate that creating the service was successful you can run:

```
$ kubectl get services
NAME       TYPE        CLUSTER-IP       EXTERNAL-IP    PORT(S)           AGE
jenkins    NodePort    10.103.31.217    <none>         8080:32664/TCP    59s
```

### [Access Jenkins dashboard](#access-jenkins-dashboard)
So now we have created a deployment and service, how do we access Jenkins?

From the output above we can see that the service has been exposed on port 32664. We also know that because the service is of type NodeType the service will route requests made to any node on this port to the Jenkins pod. 
With docker on Linux you can simply send traffic to the node IPs from the host without this, but to cover macOS and Windows you'll want to use these.

You may also want to see the [Extra Port Mappings](https://kind.sigs.k8s.io/docs/user/configuration/#extra-port-mappings) and [Ingress Guide](https://kind.sigs.k8s.io/docs/user/ingress)

To find the name of the pod, enter the following command:

```
$ kubectl get pods
```
or
```
kubectl get pod -l app=jenkins -o jsonpath='{.items[0].metadata.name}'
```
or
```
kubectl get --no-headers=true pods -o name | awk -F "/" '{print $2}'
```

and then you can forwarding port using the name of pod and port

```
kubectl -n jenkins port-forward $(kubectl -n jenkins get pod -l app=jenkins -o jsonpath='{.items[0].metadata.name}') 9090:8080
```

Once you locate the name of the pod, use it to access the pod’s logs.

```
$ kubectl logs <pod_name> -n jenkins
```

The password is at the end of the log formatted as a long alphanumeric string:
```
*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required.
An admin user has been created and a password generated.
Please use the following password to proceed to installation:

94b73ef6578c4b4692a157f768b2cfef

This may also be found at:
/var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```

### Post-installation setup wizard

After downloading, installing and running Jenkins using one of the procedures above, the post-installation setup wizard begins.

This setup wizard takes you through a few quick "one-off" steps to unlock Jenkins, customize it with plugins and create the first administrator user through which you can continue accessing Jenkins.

## Unlocking Jenkins

When you first access a new Jenkins instance, you are asked to unlock it using an automatically-generated password.

1. Browse to http://localhost:9090 (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears.

    ![image info](./img/setup-jenkins-01-unlock-jenkins-page.jpg)
    
2. From the Jenkins console log output, copy the automatically-generated alphanumeric password (between the two sets of asterisks).
    ![image info](./img/setup-jenkins-02-copying-initial-admin-password.png)

3. On the **Unlock Jenkins** page, paste this password into the **Administrator password** field and click **Continue**.
Notes:


You can always access the Jenkins console log from the Docker logs.

The Jenkins console log indicates the location (in the Jenkins home directory) where this password can also be obtained. This password must be entered in the setup wizard on new Jenkins installations before you can access Jenkins’s main UI. This password also serves as the default admininstrator account’s password (with username "admin") if you happen to skip the subsequent user-creation step in the setup wizard.


**You have successfully installed Jenkins on your Kubernetes cluster and can use it to create new and efficient development pipelines.**


