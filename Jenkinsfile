def project = 'jenkinsworld-demo'
def appName = 'gceme'
def feSvcName = "${appName}-frontend"
def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

/*
// generate stages for each language
// (technique adapted from https://stackoverflow.com/questions/46894308/)
def langs = ['en_US','fr_FR','es_MX','zh_CN','ru_RU']

def parallelStagesMap = langs.collectEntries {
	["${it}" : generateStage(it,feSvcName,imageTag)]
}

def generateStage(lang,feSvcName,imageTag) {
	return {
		stage("test language: ${lang}") {
      agent{
        container('kubectl') {}
      }
      steps{
        echo "create service: ${lang}"
        // Create namespace if it doesn't exist
        sh("kubectl get ns ${env.BRANCH_NAME}-${lang.replace('_','-').toLowerCase()} || kubectl create ns ${env.BRANCH_NAME}-${lang.replace('_','-').toLowerCase()}")
        // Don't use public load balancing for development branches
        sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
        sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/dev/*.yaml")
        sh("sed -i.bak 's#en_US#${lang}#' ./k8s/dev/*.yaml")
        // TODO: wait until apply is complete before moving on
  sh("kubectl --namespace=${env.BRANCH_NAME}-${lang.replace('_','-').toLowerCase()} apply -f k8s/services/")
        sh("kubectl --namespace=${env.BRANCH_NAME}-${lang.replace('_','-').toLowerCase()} apply -f k8s/dev/")
        echo "To access your environment run `kubectl proxy` then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}-${lang.replace('_','-').toLowerCase()}/services/${feSvcName}:80/"					
        echo "TODO: verify"
			}
		}
	}
}
*/


pipeline {
  agent {
    kubernetes {
      label 'sample-app'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
labels:
  component: ci
spec:
  # Use service account that can deploy to all namespaces
  serviceAccountName: cd-jenkins
  containers:
  - name: golang
    image: golang:1.10
    command:
    - cat
    tty: true
  - name: gcloud
    image: gcr.io/cloud-builders/gcloud
    command:
    - cat
    tty: true
  - name: kubectl
    image: gcr.io/cloud-builders/kubectl
    command:
    - cat
    tty: true
  - name: kubectl2
    image: gcr.io/cloud-builders/kubectl
    command:
    - cat
    tty: true
  - name: kubectl3
    image: gcr.io/cloud-builders/kubectl
    command:
    - cat
    tty: true
  - name: kubectl4
    image: gcr.io/cloud-builders/kubectl
    command:
    - cat
    tty: true
  - name: kubectl5
    image: gcr.io/cloud-builders/kubectl
    command:
    - cat
    tty: true    
"""
}
  }
  stages {
    stage('Test') {
      steps {
        container('golang') {
          sh """
            ln -s `pwd` /go/src/sample-app
            cd /go/src/sample-app
            go test
          """
        }
      }
    }
    stage('Build and push image with Cloud Build') {
      steps {
        container('gcloud') {
          sh "PYTHONUNBUFFERED=1 gcloud builds submit -t ${imageTag} ."
        }
      }
    }
    stage('Test Languages') {
      // Test application in multiple language environments, in parallel
      parallel {
         stage("item 1") {
          steps{
            container('kubectl1') {
              echo "I am in item 1"
              sh("kubectl version --short ")
            }
          }
        }
        stage("item 2") {
          steps{
            container('kubectl2') {
              echo "I am in item 2"
              sh("kubectl version --short ")
            }
          }
        }
        stage("item 3") {
          steps{
            container('kubectl3') {
              echo "I am in item 3"
              sh("kubectl version --short ")
            }
          }
        }
        stage("item 4") {
          steps{
            container('kubectl4') {
              echo "I am in item 4"
              sh("kubectl version --short ")
            }
          }
        }
        stage("item 5") {
          steps{
            container('kubectl5') {
              echo "I am in item 5"
              sh("kubectl version --short ")
            }
          }
        }
      }
    }
    stage('Deploy Canary') {
      // Canary branch
      when { branch 'canary' }
      steps {
        container('kubectl') {
          // Change deployed image in canary to the one we just built
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/canary/*.yaml")
          sh("kubectl --namespace=production apply -f k8s/services/")
          sh("kubectl --namespace=production apply -f k8s/canary/")
          sh("echo http://`kubectl --namespace=production get service/${feSvcName} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${feSvcName}")
        } 
      }
    }
    stage('Deploy Production') {
      // Production branch
      when { branch 'master' }
      steps{
        container('kubectl') {
        // Change deployed image in canary to the one we just built
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/production/*.yaml")
          sh("kubectl --namespace=production apply -f k8s/services/")
          sh("kubectl --namespace=production apply -f k8s/production/")
          sh("echo http://`kubectl --namespace=production get service/${feSvcName} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${feSvcName}")
        }
      }
    }
    stage('Deploy Dev') {
      // Developer Branches
      when { 
        not { branch 'master' } 
        not { branch 'canary' }
      } 
      steps {
        container('kubectl') {
          // Create namespace if it doesn't exist
          sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
          // Don't use public load balancing for development branches
          sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/dev/*.yaml")
          sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/services/")
          sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/dev/")
          echo 'To access your environment run `kubectl proxy`'
          echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
        }
      }     
    }
  }
}