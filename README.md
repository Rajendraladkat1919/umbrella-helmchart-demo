# Umbrella helm chart Project

# 0. Problem Statement:
    
    We need some mechanism where we can do below things:

    1. Deploy multiple microservices in single go with same version of each micro services from dev to production.
    2. We need back tracing of deployed microservices in any environmnet.
    3. We need single helm chart which will deploy all microservices in single go.
    4. Promote the artifact from one environmnet to other with same set of artifact.
    5. Automate software release process with less manula intervation.

# 1. Approach to fix above problem statement.

    There are few technique to address this issue and umbrella helm chart deployment is one approach.
    
    Technique : Use Umbrella Charts
    Helm charts have the ability to include other charts, referred to as subcharts, via their dependencies section. When a chart is created for the purpose of grouping together related subcharts/services, such as to compose a whole application or deployment, we call this an umbrella chart.
    When using an umbrella chart, it is really easy to override values in its subcharts, such as the tag of a new image. We do this in the umbrella chart’s values.yaml, by placing the new value at <name-of-subchart>.path.to.image.tag. For example, to override the image.tag value in the buslog subchart our previous umbrella chart example, we would place this set of attributes into the umbrella chart’s values.yaml:
    ```
        buslog:
            image:
            tag: featureXYZ-4a28596
    ```
    So the idea here is that you would have an umbrella chart in the Git repo/branch that you use for deploying one or more services (usually a group of related services) to an environment. Then for most changes, your CI or promotion pipeline is just going to update one line in this umbrella chart’s values.yaml, and it will leave the subchart (buslog in this example) completely alone. Your subchart would only need to have its chart version incremented on those rare occasions when its non-image elements change (usually as part of a CI pipeline).

# 2. POC Immplementation details:
    
    2.1 : Prerequisite:
        
        This poc implmented on local machine for small development and testing purpose. Following tool and Technology we used while implmenting Umbrella helm chart POC.
        1. Helm
        2. Minikube
        3. Jenkins Helm Chart
        4. Required helm , Jenkins and docker plugin.
        5. Github as SCM
        6. Decelarative pipeline approach to deploy umbrealla helm chart.
    
    2.2 Chart folder structure:

        ```
            ➜  ~ tree guestbook
            guestbook
            ├── Chart.lock
            ├── Chart.yaml
            ├── Chart.yaml.bak
            ├── Jenkinsfile
            ├── README.md
            ├── charts
            │   ├── consul-10.4.0.tgz
            │   ├── ghost-17.0.6.tgz
            │   ├── kibana-10.0.6.tgz
            │   ├── mongodb-12.1.3.tgz
            │   └── redis-12.7.7.tgz
            ├── templates
            │   ├── NOTES.txt
            │   ├── _helpers.tpl
            │   ├── deployment.yaml
            │   ├── hpa.yaml
            │   ├── ingress.yaml
            │   ├── service.yaml
            │   └── serviceaccount.yaml
            └── values.yaml

            2 directories, 18 files
        ```

    2.3 How this works:

       We create one parent helm chart and other helm chart we can inlcude as dependent helm chart inside helm chart.yaml like below.
       
       ```
        appVersion: v4
        dependencies:
            - name: redis
            version: 12.7.x
            repository: https://charts.bitnami.com/bitnami
            - name: ghost
            version: 17.0.6
            repository: https://charts.bitnami.com/bitnami
            - name: consul
            version: 10.4.0
            repository: https://charts.bitnami.com/bitnami
            - name: kibana
            version: 10.0.6
            repository: https://charts.bitnami.com/bitnami
            - name : mongodb
            version: 12.1.3
            repository: https://charts.bitnami.com/bitnami
        ```  
        Then we perform 
        ```
        `helm dependency update`

        ```
        This commnd lock the respective helm chart with version inside `Chart.lock` and download the required dependendent helm chart inside charts folder.
        ```
        ➜  guestbook git:(main) ✗ tree charts
        charts
        ├── consul-10.4.0.tgz
        ├── ghost-17.0.6.tgz
        ├── kibana-10.0.6.tgz
        ├── mongodb-12.1.3.tgz
        └── redis-12.7.7.tgz
        ```

        Once this stpes finshed we can package and publish the umbrella helm chart to git repo or artifactory.

        To package the helm chat.

        `helm pakage .`
        
        This command will package the helm chart and make tgz file as artifact.

        `helm publish ` 

        This command will publish helm chart to helm repo either on git or artifactory or s3.

        Once aabove steps completed we install the umbrella helm chart using below command.

        `helm install <release name> <chart>  <namespae>`


# 3. We implmented this using Jenkisfile with below stage.

        ```
                        pipeline {
                agent {
                    kubernetes {
                    label 'helm-pod'
                    serviceAccount 'jenkins'
                    containerTemplate {
                        name 'helm-pod'
                        image 'dtzar/helm-kubectl'
                        ttyEnabled true
                        command 'cat'
                    }
                    }
                }
                /*environment {
                        tag = sh(returnStdout: true, script: "git rev-parse --short=10 HEAD").trim()
                    }*/
                stages {
                    stage('Helm Dependancy Update..') {
                    steps {
                        container('helm-pod') {
                        git url: 'https://github.com/rladkat/guestbook.git', branch: 'main'
                        sh '''
                            
                            helm dependency update
                            if [ $? -eq 0 ]; then
                                echo "dependency update successfully completed."
                            else
                                echo "dependency update failed. Halt the package build process."
                            fi
                        '''
                        }
                    }
                    }

                    stage('Helm Package started') {
                    steps {
                        container('helm-pod') {

                            //echo "The build number is ${env.BUILD_NUMBER}-${env.GIT_COMMIT} "
                            //sed -i.bak "s/^version:/version: $BUILD_NUMBER/" Chart.yaml
                            sh '''
                                echo "I can access $BUILD_NUMBER in shell command as well." | cut -f4 -d' '
                                
                                helm package .
                                if [ $? -eq 0 ]; then
                                    echo "Helm packaging completed."
                                else
                                    echo " Helm pckaging failed."
                                fi
                                mv *.tgz guestbook-$BUILD_NUMBER.tgz
                                ls -al
                            '''
                        }
                    }
                    }
                    stage('Helm publish') {
                    steps {
                        container('helm-pod') {

                        echo "Pulishing helm chart to artifactory. "
                            
                        }
                    }
                    }

                    stage('Helm Deploy') {
                    steps {
                        container('helm-pod') {

                        echo "Deploying on minikube."
                        sh 'helm upgrade --install my-book ./*.tgz'
                        }
                    }
                    }
                    stage('List Helm Deployment') {
                    steps {
                            withKubeConfig([credentialsId: 'xxxx', serverUrl: 'xxxx']) {
                            sh 'helm ls'
                        }
                    }
                    }

                    stage('Microservices manifest') {
                    steps {
                        container('helm-pod') {

                        echo "Below is the deployed version of microservices on k8s cluster with image name and docker tag. "
                        sh 'kubectl get po -n jenkins -o yaml | grep image: | cut -d ":" -f2,3 | sort | uniq >> deployedservices.txt'
                        sh 'cat deployedservices.txt'
                        
                        echo "Uploading manifeast on artifactory."
                            
                        }
                    }
                    }
                }
                }
```
    
After  succesully deployment it will gives details which services deployed with which docker tag just for information purpose.

1. Helm release details :

```
    NAME      	NAMESPACE	REVISION	UPDATED                               	STATUS  	CHART          	APP VERSION
    my-gb     	default  	1       	2022-05-09 19:06:41.287918 +0530 +0530	deployed	guestbook-0.1.0	v4 
```
2. Deplyed services details:

```
    + cat deployedservices.txt
    bitnami/redis:6.0.11-debian-10-r0
    docker.io/bitnami/redis:6.0.11-debian-10-r0
    dtzar/helm-kubectl:
    dtzar/helm-kubectl:latest
    gcr.io/google-samples/gb-frontend:v4
    jenkins/inbound-agent:4.11-1-jdk11
    jenkins/jenkins:2.332.2-jdk11
    kiwigrid/k8s-sidecar:1.15.0
```