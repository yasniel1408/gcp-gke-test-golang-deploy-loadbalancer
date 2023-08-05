pipeline {

  environment {
    PROJECT = "YOUR_PROJECT"
    APP_NAME = "valkyrie-app"
    FE_SVC_NAME = "valkyrie-dev"
    CLUSTER = "valkyrie-dev"
    CLUSTER_ZONE = "us-east1-d"
    IMAGE_TAG = "gcr.io/${PROJECT}/${APP_NAME}:dev.${env.BUILD_NUMBER}"
    JENKINS_CRED = "${PROJECT}"
  }

  agent {
    kubernetes {
      label 'valkyrie-app'
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
"""
}
  }
  stages {
    stage('Test') {
      steps {
        container('golang') {
          sh """
            ln -s `pwd` /go/src/va
            cd /go/src/va/source
            go test
          """
        }
      }
    }
    stage('Build and push image with Container Builder') {
      steps {
        container('gcloud') {
          sh "PYTHONUNBUFFERED=1 gcloud builds submit -t ${IMAGE_TAG} ."
        }
      }
    }
    stage('Deploy') {
      steps{
        container('kubectl') {
        // Change deployed image in canary to the one we just built
          sh("sed -i.bak 's#gcr.io/${PROJECT}/valkyrie-app:v0.0.1#${IMAGE_TAG}#' ./k8s/deployment.yaml")
          step([$class: 'KubernetesEngineBuilder',namespace:'default', projectId: env.PROJECT, clusterName: env.CLUSTER, zone: env.CLUSTER_ZONE, manifestPattern: 'k8s', credentialsId: env.JENKINS_CRED, verifyDeployments: true])
          sh("echo http://`kubectl --namespace=default get service/${FE_SVC_NAME} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${FE_SVC_NAME}")
        }
      }
    }
  }
}