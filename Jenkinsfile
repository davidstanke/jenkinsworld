def project = 'jenkinsworld2018'
def appName = 'gceme'
def feSvcName = "${appName}-frontend"
def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

// generate stages for each language
// (technique adapted from https://stackoverflow.com/questions/46894308/)
def langs = ['en_US','fr_FR','es_MX','zh_CN','ru_RU']

def parallelStagesMap = langs.collectEntries {
	["${it}" : generateStage(it,feSvcName,imageTag)]
}

def generateStage(lang,feSvcName,imageTag) {
    def name_tag = "${env.BRANCH_NAME}-${lang.replace('_','-').toLowerCase()}-${env.BUILD_NUMBER}"
	return {
		stage("test language: ${lang}") {
			stage("create service") {
	            echo "create service: ${lang}"
	            container('kubectl') {

					// Create namespace if it doesn't exist
		            sh("kubectl get ns ${name_tag} || kubectl create ns ${name_tag}")
		            // Don't use public load balancing for development branches
		            sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
		            sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/dev/*.yaml")
		            sh("sed -i.bak 's#en_US#${lang}#' ./k8s/dev/*.yaml")
		            // TODO: wait until apply is complete before moving on
					sh("kubectl --namespace=${name_tag} apply -f k8s/services/")
		            sh("kubectl --namespace=${name_tag} apply -f k8s/dev/")

				}
			}
			stage("verify service") {
				echo "TODO: test language"
			}
            stage("teardown resources") {
                container('kubectl') {
                    sh("kubectl delete ns ${name_tag}")
                }
            }
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
  triggers {
    // build every 20 minutes to keep a baseline load
    cron('H/20 * * * *')
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
        // when { branch 'experimental'}
        steps {
			script {
				parallel parallelStagesMap
			}
		}
    }
    stage('Deploy Production') {
      // Production branch
      when { branch 'master' }
      steps{
        container('kubectl') {
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/production/*.yaml")
          sh("kubectl --namespace=production apply -f k8s/services/")
          sh("kubectl --namespace=production apply -f k8s/production/")
          sh("echo http://`kubectl --namespace=production get service/${feSvcName} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${feSvcName}")
        }
      }
    }
  }
}
