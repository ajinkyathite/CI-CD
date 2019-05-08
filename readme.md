CI/CD with Jenkins, Docker and Kubernetes
=
CI/CD stands for Continuous Integration and Continuous Delivery. It is a set of software development operating principles that enable teams to deliver code changes. The main CI/CD concept is continuously making these small changes to the code, building, testing and delivering more often, more quickly and more efficiently.

Visual depiction of CI/CD: 
![alt text][logo]

Ideal Pipeline flow
-
![alt text][SLDC]

Above Pipeline is a general Software Development route in given Software Development Life Cycle (SDLC). Tools might be different per your needs or stages might not apply to your needs or additional stage could be added to the pipeline.

Setting up Jenkins
-
Once the Jenkins image has been built, we create an instance from that image. To log into the instance the first time, you will need to get the one time password by sshing into the instance and running this command.

    sudo cat /tmp/jenkins/secrets/initialAdminPassword
    
#### 1. Enable automatic builds
On the landing page, on the left hand side, you will see a menu. Under that menu, select Manage Jenkins, then Configure System. On the new page, find the Github section and click on Advanced. You should see a checkbox, click on it. The input field should be auto-populated with a URL.
That url is a webhook which should be added to the github repository’s Settings as a service, under Jenkins (Github plugin), in the Integrations and Services section. This is to enable automatic builds on every commit.
For this to work, make sure you have set a domain name. for some reason the webhook does not work with an IP in the URL. If you don’t have a domain, you will have to manually trigger the builds.

#### 2. Managing secrets
Head back over to the menu on your left and click on Manage Jenkins again. Close to the bottom of that menu you will see a Credentials option, click on it, then under the Stores scoped to Jenkins click on (global). Here is where we save all the values we do not want to expose, also know as SECRETS. Since we are working with Docker we will need to login to it during the deployment pipeline so we add the docker password here as a secret text. It is one of the available options under kind after you click on the Add credentials button. 
For our pipeline, this is the only secret we have, however there will be cases where you have several variables to hide and this is where you add them. Use IDs that match the variable names you have in place of where the values should be.

#### 3. Add the project management tool
Back to Manage Jenkins, under Global Tool Configuration , scroll down till you find the section with Maven. Click on the Maven Installation button and enter a name which you will use to reference the version of Maven to use, I used M3. The reason for this configuration is because Maven is the project management tool which was used by our microservice software developer.

#### 4. Install necessary plugins
Back to `Manage Jenkins`, we click on the `Manage Plugins` section and under available, we install the [Docker](https://wiki.jenkins.io/display/JENKINS/Docker+Plugin) and [Blue Ocean](https://wiki.jenkins.io/display/JENKINS/Blue+Ocean+Plugin) plugins. Docker for Jenkins to be able to use a Docker host to dynamically provision build agents, run a single build, then tear-down agent while Blue Ocean is a more user friendly interface for working with Jenkins Pipelines. You should restart Jenkins once the installs are done.

Setting up the pipeline as code
-
We will use Blue Ocean to create the pipeline but for that to happen, there should a _Jenkinsfile_ in the repository. So we head over to the repository and add the file with at least one stage(see below for the stages) and pushing it before going back to the Jenkins Blue Ocean dashboard to setup the pipeline.

**Jenkinsfile**
```
node {
  stage('SCM checkout') {
    git 'https://github.com/Thegaijin/microservice-kubernetes.git'
  }

  stage('test') {
    def mvnHome = tool name:'M3', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn"
    sh "${mvnCMD} clean test"
  }

  stage('compile and package') {
    def mvnHome = tool name:'M3', type: 'maven'
    def mvnCMD = "${mvnHome}/bin/mvn"
    sh "${mvnCMD} clean package"
  }

  stage('Build docker images') {
    sh 'docker-compose build'
  }

  stage('login to dockerhub') {
    withCredentials([string(credentialsId: 'docker-pwd', variable: 'dockerhubpwd')]) {
      sh 'docker login -u thegaijin -p ${dockerhubpwd}'
    }
  }

  stage('Push images to dockerhub') {
    sh 'docker-compose push'
  }

  stage('deploy') {
    sh 'kubectl apply -R -f ./k8s'
  }
}
```
The final stage, deploy, also requires a **handshake** between Jenkins and the cloud infrastructure and this is established when the Kubernetes cluster is created. This is established by creating a .kube directory in the Jenkins directory, copying the cluster config file to that directory, changing the file owner to the Jenkins user and then making it executable. This enables Jenkins to execute the kubectl commands.

```
sudo mkdir -p /var/lib/jenkins/.kube
sudo cp ~/.kube/config /var/lib/jenkins/.kube/
cd /var/lib/jenkins/.kube/
sudo chown jenkins:jenkins config
sudo chmod 750 config
```

[logo]: CICD.png "Visual depiction of CI/CD"
[SLDC]: SDLC.png "Ideal Jenkins Pipeline FlowD"

---
More info: [Pipeline as a Code using Jenkins](https://medium.com/@maxy_ermayank/pipeline-as-a-code-using-jenkins-2-aa872c6ecdce)
