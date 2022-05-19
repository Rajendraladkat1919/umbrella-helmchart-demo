pipeline {
  agent {
    kubernetes {
      label 'helm-pod'
      serviceAccount 'jenkins-helm'
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
          git url: 'https://github.com/rladkat/helm-deployment-poc.git', branch: 'main'
          sh '''
            helm repo update
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
        sh '''
          helm upgrade --install my-gb ./*.tgz -n jenkins
        
        '''

        }
      }
    }
    
    stage('List Helm Deployment') {
      steps {
            //withKubeConfig([credentialsId: 'kubeconfig', serverUrl: 'https://192.168.64.3:8443']) {
            sh 'helm ls -A'
            
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
