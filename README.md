# Needs validation ** 
# Jenkins CI/CD on GuestCluster 
### Test cases guide for VS7-K8s { Project Pacific } 
### Install steps go in ORDER

### Sumamry of Dev experiencce 

This repo will take you through Installing Jenkins on your Env and run SpringBoot CI/CD Pipeline to build a Java Spring Boot app, test the app with maven unit tests, create a jar and then upload the Jar into a Git repo. It will then take  the Docker image and deploy it to  a K8 cluster. 

We will be deploying the SpringBoot app into a "default" namespace. In the pipeline  will also use Config's for Kubernetes and Github and use them via plugins.

#### Pre-Req : 

- Need Persistant Volume : We use "projectpacific-storage-policy" in this example.
- K8 cluster to install Master and Slave Pods.
- All yaml under the JeninsKinstall folder in this repo 


### Clone the repo  ** 
`$ git clone https://https://github.com/reagul/Jenkins-For-ProjectPacific` \
`$ cd kubernetes-jenkins`

### Create the project Namespace
`$ kubectl create ns jenkins` 

### Switch context to jenkins namespace
`$ kubectl config set-context $(kubectl config current-context) --namespace=jenkins`

### Prepare Persistence Storage
#### Storage Class is already configured for this GuestCluster. Here we provision a Claim for 4 GB. This claim will be applied to the Jenkins Deployment yaml.

`$ cat jenkins-sc.yaml` **
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  storageClassName: projectpacific-storage-policy
```


`$ kubectl apply -f jenkins-pvc.yaml` \

### Inspect the PVC / SC to validate bound Volume

`$ kubectl get sc`

`$ kubectl get pvc`


### Install RBAC for K8 use so it can spin master and slave pods

This Jenkins master will initiate slave pod creation for slaves on Kubernetes.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]

```

`$ kubectl apply -f jenkins.rbac.yaml sc`


### Jenkins Deployment 

The default jenkins/jnlp-slave image does not contain the kubectl or helm binaries, so you will need to build a custom image to use the kubernetes-cli plugin. Afterwards, push it to a registry \

`$ kubectl apply -f ** ` \


#### Configure Jenkins Service / Ingress. This allows you to access Jenkins over URL.

`$ kubectl apply -f jenkins.service.yaml`

This will initate a LoadBalancer type ingress and once applied will show up like this 

```
$ kubectl get svc 

NAME                         TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                                       AGE
hello-spring-boot-actuator   LoadBalancer   198.57.146.34    10.10.20.153   80:32303/TCP                                  11h
jenkins                      LoadBalancer   198.60.149.135   10.10.20.145   8080:31403/TCP,50000:32715/TCP,80:32645/TCP   5d19h

```


You might additionally enter the LB Eternal Ip into your local DNS.


### Jenkins First time Acess and Password

Once you access the Jenkins URL, you will be prompted for a TMP password which you can fetch by doing

`kubectl logs jenkinsmasterpod2adsvadf`


### Jenkins Slave 

You can use the supplied Jenkins slave as is but `recommend` you clone it to YOUR local repository and version/tag it before using. 
In case you want to  run custom commands inside the slave, which you might do with custom scripts being used in pilepile, then you might need to extend/custom build the avaialble slave images on Dockerhub. The process involves taking the slave Dockerfile and installing custom software modules and creating a new image.

Let us configure a Kubernetes cloud inside Jenkins.

### Kubernetes Cloud config values


This Jenkins master will initiate slave pod creation so we are configuring a Kubernetes cloud config with our custom slave images in it.

In this test-case we will configure a slave for Kubenretes Cloud \

* Go to Manage Jenkins | Bottom of Page | Cloud | Kubernetes (Add kubenretes cloud)
* Fill out plugin values
    * Name: kubernetes
    * Kubernetes URL: https://kubernetes.default:443
    * Kubernetes Namespace: jenkins
    * Credentials | Add | Jenkins (Choose Kubernetes service account option & Global + Save)
    * Test Connection | Should be successful! If not, check RBAC permissions and fix it!
    * Jenkins URL: http://jenkins
    * Tunnel : jenkins:50000
    * Apply cap only on alive pods : yes!
    * Add Kubernetes Pod Template
        * Name: jenkins-slave
        * Namespace: jenkins
        * Labels: jenkins-slave (you will need to use this label on all jobs)
        * Containers | Add Template
            * Name: jnlp
            * Docker Image: aimvector/jenkins-slave
            * Command to run : <Make this blank>
            * Arguments to pass to the command: <Make this blank>
            * Allocate pseudo-TTY: yes
            * Add Volume
                * HostPath type
                * HostPath: /var/run/docker.sock
                * Mount Path: /var/run/docker.sock
        * Timeout in seconds for Jenkins connection: 300
* Save


### Installing Kubernetes plugin

Select **Manage Jenkins**>**Manage Plugins**. Select the **Available** tab and enter "kubernetes" in the search filter. Choose the following plugin options:
	
- Kubernetes Continuous Deploy
- Kubernetes Cli
- Kubernetes Credentials Provider
- Kubernetes :: Pipeline :: DevOps Steps
- Kubernetes :: Pipeline :: Kubernetes Steps

Select **Download now and install after restart** \
On the following page, choose **Restart Jenkins when installation is complete and no jobs are running**

After a few minutes, if the status does not change, select **Return to Dashboard** and complete the login prompt. An ENABLE AUTO REFRESH link is avaible in the upper right of the window but the results aren't consistant across browsers. Again, if after waiting a few minutes for auto-refresh to update, replace to the current brower URL with the welcome page URL, `http://<HostName>/<JenkinsUriPrefix>`

> Before proceeding, consider returning to the Plugin Manager, searching for "docker" in the list of available plugins and repeating the previous process for installing plugins: "docker-build-step" and "Docker"


### Running Jenkins Pipelines.

There are code samples under the JenkinsScriptFiles that can be used to run some sample pipelines. 

### Configuring the Kubernetes Credentials and Plugin

From the Jenkins Dashboard select **Credentials**>**System**>**Global credentials (unrestricted)**>**Add Credentials** \
Select the *Kind* drop-down and choose **Kubernetes Service Account**

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/kubernetes-service-account.png)

Select **OK**

Again, Select **Add Credentials** \
Select the *Kind* drop-down and choose **Kubernetes configuration (kubeconfig)**

Open a command-line and make the following script executable, \
`$ chmod +x exist-sa_kubecfg.sh` \
Then, execute the script followed by the service account name and namespace name. In this case, "jenkins" for both values \
`$ ./exist-sa_kubecfg.sh jenkins jenkins` \
The script creates a new file in the present directory, named jenkins-sa.kubeconfig.  With a text-editor compare the contents to the output from `$ kubectl config view` and verify the context, server, and cluster values match.

Return to the Jenkins Web UI, where you left off configuring the **Kubernetes configuration (kubeconfig)** credentials. Enter a *description* such as **Jenkins Service Account Kubeconfig**. Select the radio button for **Enter Directly**. Paste the content from the updated *jenkins-sa.kubeconfig* file

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/sa-kubeconfig.png)

Select **Save**

Return to the Jenkins Dashboard and select **Manage Jenkins**>**Configure System**. Scroll to the section *Cloud*>*Kubernetes* and notice the required fields.

Open the command-line and issue the command \

`$ kubectl config view` \

Copy the URL from the server parameter output and replace the form value for *Kubernetes URL*
```
apiVersion: v1
clusters:
- cluster:
    server: https://pksk8s01api.lab.local:8443

hY8kAGeuT10K0cOVwtvsVXurIByTVettpfKOK3vMn5y5CXRtKXzJHYs8F25wojXs
1Jczjvdbfnw4miV4fK8D

```
Select the *Credentials* drop-down option and choose **Secret Text** 

Select **Test Connection** and test results should indicate **Connection test successful** 

![alt text](https://github.com/csaroka/kubernetes-jenkins/blob/master/images/kubernetes-cloud-config.png)

For basic operations, all other fields' default value should've populated with input from the values.yaml at the time of deployment. Verify the following:

- *Cloud*/*Kubernetes*/*Jenkins URL* = **http://someip:8080*

### Using KubeConfig Plugin creds 


This code is pulling the K8 configs and using it. We get the ```kubeconfigId ``` from whenn we created the Plugin.

``` stage('Deploy App') {
      steps {
        script {
          kubernetesDeploy(configs: "hello-springboot-actuator.yaml", kubeconfigId: "pacifickube")
        }
      }
    }
```


### Configuring Github Creds

When the Pipeline pulls / pushes into GitHub repo, we will use a Jenkins Plugin to apply these creds into the pipeline code. 

From the 


### Using GitHub creds 

This code snippet shows the way to use Giuthub creds from the plugin. We get the credentialsId from when we create the username and password plugin. 

```
stage("Git creds"){
            steps {
            withCredentials([usernamePassword(credentialsId: 'newpaulgitpwd', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                sh("""
               
                    git config --global user.email "paul@paul.com"
                    git config --global user.name ${GIT_USERNAME}
                    git config --local credential.helper "!f() { echo username=paul; echo password=${GIT_PASSWORD}; }; f"

```

### Spring Boot app Deployed 

You can test the Spring boot app once deployed with ths example REST call. You can do this on a browser or with Curl.

```
$ curl http://10.10.20.153/hello-world

Returns

{"id":4,"content":"Hello, Stranger!"}
```
