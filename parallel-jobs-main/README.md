# parallel-jobs


### Helm deployment of jenkins


* Add all the helm repo to your system

```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

### Create an ingress controller

```shell
kubectl create ns ingress-controller

helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace=ingress-controller \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```

* Get the LoadBalancer IP to access Ingress controller

```shell
kubectl -n ingress-controller get svc ingress-nginx-controller -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
```

* Install Cert Manager

```shell
kubectl create ns cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace=cert-manager --version v1.1.0
```

* Create a CA ClusterIssuer

Please update email: `mymail@gmail.com` part with your mail id in production.

```code
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: mymail@gmail.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - http01:
       ingress:
         class: nginx
```

```shell
kubectl apply -f ./certManagerCI_production.yaml
```

* Test the certificate issuer to test the webhook works okay

```shell
kubectl apply -f test-resources.yaml
kubectl describe certificate -n cert-manager-test
kubectl delete -f test-resources.yaml
```


### Configure Kubernetes Cluster and Create node-pools with the following labels

```code
server-use: server (dedicated instance)
server-use: agent (preemtable instance)
```

* Take a look into these parameters in `values.yaml` file

```code
controller
  adminUser
  serviceType
  loadBalancerSourceRanges
  additionalPlugins
  JCasC
  nodeSelector
  ingress
  httpsKeyStore

agent
  namespace
  image
  resources
  nodeSelector

backup
```

* update the github organisation's OAuth App (optional, needed if you are using the Github OAuth)

```code
https://jenkins_url/securityRealm/finishLogin
```

* deploy jenkins

```shell
helm -n jenkins install jenkins-server jenkins/jenkins -f jenkins-values.yaml
```

* `ClusterIP` can access from localhost

```shell
kubectl port-forward service/jenkins-server 8080:8080
```

[ref](https://github.com/jenkinsci/helm-charts/tree/main/charts/jenkins)


### Jenkins Installation (master or slave need to configure accordingly)

[ref](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)

[jenkins slave on gcp](https://cloud.google.com/solutions/using-jenkins-for-distributed-builds-on-compute-engine)

```shell
# dependencies for jenkisnfile stages
apt-get update
apt-get install wget gettext jq apt-transport-https ca-certificates curl gnupg-agent software-properties-common -y
# java installation
apt-get install openjdk-11-jdk -y
java -version
# docker installation
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io -y
docker version
# kubectl install (1.16 version here)
curl -LO https://dl.k8s.io/release/v1.16.0/bin/linux/amd64/kubectl
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
# jenkins installation
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
apt-get update
apt-get install jenkins -y
systemctl start jenkins
systemctl status jenkins
# grant permission for jenkins group
usermod -aG docker jenkins
```

### ssl termination

```shell
# Self-Signed SSL Certificate
openssl req -newkey rsa:4096 \
            -x509 \
            -sha256 \
            -days 365 \
            -nodes \
            -out jenkins.crt \
            -keyout jenkins.key \
            -subj "/C=IN/ST=Karnataka/L=Bangalore/O=Security/OU=IT Department/CN=www.jenkins.local"

# Convert SSL keys to PKCS12 format
openssl pkcs12 -export -out jenkins.p12 \
    -passout 'pass:your-secret-password' \
    -inkey jenkins.key \
    -in jenkins.crt \
    -name www.jenkins.local

# Convert PKCS12 to JKS format
keytool -importkeystore -srckeystore jenkins.p12 \
    -srcstorepass 'your-secret-password' \
    -srcstoretype PKCS12 \
    -srcalias www.jenkins.local \
    -deststoretype JKS \
    -destkeystore jenkins.jks \
    -deststorepass 'your-secret-password' \
    -destalias www.jenkins.local

# create a keystore and save the JKS file
mkdir /var/lib/jenkins/keystore
cp jenkins.jks /var/lib/jenkins/keystore/
chown -R jenkins: /var/lib/jenkins/keystore
chmod 700 /var/lib/jenkins/keystore
chmod 600 /var/lib/jenkins/keystore/jenkins.jks
```

# Modify Jenkins Configuration for SSL

```shell
vim /etc/default/jenkins
```

```code
HTTP_PORT=-1
JENKINS_ARGS="--webroot=/var/cache/$NAME/war --httpsPort=8443 --httpsKeyStore=/var/lib/jenkins/keystore/jenkins.jks --httpsKeyStorePassword=your-secret-password --httpPort=$HTTP_PORT"
```

[ssl ref](https://wiki.wocommunity.org/display/documentation/Installing+and+Configuring+Jenkins)

[ssl ref 1](https://devopscube.com/configure-ssl-jenkins/)



### Automatically Generating Jenkins Jobs (Jenkins Job Builder)

#### Install and Configure

```shell
# install
apt-get install python3-pip --fix-missing
pip3 install --user jenkins-job-builder
```

```shell
# configure
vim jenkins_jobs.ini
```

```code
[job_builder]
ignore_cache=True
keep_descriptions=False
recursive=False
allow_duplicates=False
update=all

[jenkins]
user=custom_user
password=token
url=https://localhost:8443
query_plugins_info=False
```


#### job template

```code
- job-template:
    # default variable
    disable_job: false
    job_description: 'Do not edit this job through the web....!!!'
    trigger_branch: 'master,main,develop'
    git_repo_url: git@github.com:mytest/jobs.git
    git_credentials_id: 'jenkins-credential-id'
    job_folder: 'cicd_pipeline'
    # job configuration
    name: '{job_folder}/{customer}'
    id: 'branch-pipeline'
    description: '{job_description}'
    display-name: '{customer}'
    periodic-folder-trigger: '1m'
    project-type: multibranch
    number-to-keep: '10'
    script-path: Jenkinsfile
    disabled: '{disable_job}'
    scm:
      - git:
          url: '{git_repo_url}'
          credentials-id: '{git_credentials_id}'
          discover-tags: false
          build-strategies:
            - all-strategies-match:
                strategies:
                  - regular-branches: true
          property-strategies:
            named-branches:
              defaults:
                - suppress-scm-triggering: true
              exceptions:
                - exception:
                    branch-name: '{trigger_branch}'
                    properties:
                      - suppress-scm-triggering: false

- job:
    name: cicd_pipeline
    project-type: folder
```


#### job profiles

This job profiles will use the above job template for the job creation, make sure that you add the job profile and job template in a single file : `Jenkinsjob`


```code
- project:
    name: my_project
    customer:
        - ubuntu:
            disable_job: true
            trigger_branch: 'master,main,develop'
            job_description: 'pipeline job for ubuntu deployment'
        - centos
        - redhat
        - debian
    jobs:
        - branch-pipeline
```




```shell
# create jenkins job
jenkins-jobs --conf keystore/jenkins_jobs.ini update jenkins-jobs/Jenkinsjob

# update / create new jobs and delete old jobs
jenkins-jobs --conf keystore/jenkins_jobs.ini update jenkins-jobs/Jenkinsjob --delete-old
```



[ref 1](https://medium.com/slalom-build/automatically-generating-jenkins-jobs-d30d4b0a2b49)

[Jenkins Job Builder install & config official](https://jenkins-job-builder.readthedocs.io/en/latest/installation.html)

[Jenkins Job Builder definition official](https://docs.openstack.org/infra/jenkins-job-builder/definition.html)

[Job DSL Plugin official](https://jenkinsci.github.io/job-dsl-plugin/#path/folder)

[Job DSL Plugin ref 1](https://www.digitalocean.com/community/tutorials/how-to-automate-jenkins-job-configuration-using-job-dsl)

[Job DSL Plugin ref 2](https://metadrop.net/en/articles/jenkins-and-job-dsl-plugin)