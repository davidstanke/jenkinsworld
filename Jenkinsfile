def project = 'jenkinsworld-demo'
def appName = 'gceme'
def feSvcName = "${appName}-frontend"
def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

// generate stages for each language
// (technique adapted from https://stackoverflow.com/questions/46894308/)
def langs = ['en_US','fr_FR']

def parallelStagesMap = langs.collectEntries {
	["${it}" : generateStage(it,feSvcName,imageTag)]
}

def generateStage(lang,feSvcName,imageTag) {
	return {
		stage("run as language: ${lang}") {
            echo "testing ${lang}"
            container('kubectl') {
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
			}
		}
		stage("verify language: ${lang}") {
			echo "TODO: verify"
		}
	}
}



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
  - name: ubuntu
    image: ubuntu
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
        when { branch 'experimental'}
        steps {
			script {
				parallel parallelStagesMap
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