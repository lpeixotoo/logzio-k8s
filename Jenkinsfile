#!/usr/bin/env groovy
@Library('nynja-common') _

def prompt(msg) {
    try{
        timeout(time: 168, unit: 'HOURS') {
            try {
                input msg
                echo "User gave approval"
                return true
            } catch (ignored) {
                echo "User pressed NO"
                return false
            }
        }
    } catch (ignored){
        echo "Nothing is pressed and prompt timed out"
        return false
    }
}

pipeline {
  environment {
    SLACK_CHANNEL = "#nynja-devops-feed"
    // Application namespace (k8s namespace, docker repository, ...)
    NAMESPACE = "logging"
    // Application name
    APP_NAME = "nccs-focus"
    // BRANCH_NAME="development"
    BRANCH_NAME="${BRANCH_NAME ? BRANCH_NAME :'development'}"
    IMAGE_NAME = "eu.gcr.io/nynja-ci-201610/${NAMESPACE}/${APP_NAME}"
    IMAGE_TAG = "${BRANCH_NAME == 'master' ? 'latest' : 'latest-' + BRANCH_NAME}"
    IMAGE_BUILD_TAG = "$BRANCH_NAME-$BUILD_NUMBER"
    // The branch to be deployed in the dev cluster, following gitflow, it should normally be "develop" or "dev"
    DEV_BRANCH = "master"
  }
  agent {
    kubernetes {
      cloud 'nynja-ci-k8s'
      label "jenkins-calling-${UUID.randomUUID().toString()}"
      yaml """
           apiVersion: v1
           kind: Pod
           spec:
             containers:
             - name: kubectl
               image: lachlanevenson/k8s-kubectl:latest
               command:
               - cat
               tty: true
             - name: helm
               image: lachlanevenson/k8s-helm:latest
               command:
               - cat
               tty: true
             - name: calling
               image: openjdk:11-jdk
               imagePullPolicy: Always
               command:
               - cat
               tty: true
               volumeMounts:
               - mountPath: /var/run/docker.sock
                 name: docker-sock
               - mountPath: /usr/bin/docker
                 name: docker-bin
             volumes:
             - name: docker-sock
               hostPath:
                 path: /var/run/docker.sock
               type: File
             - name: docker-bin
               hostPath:
                 path: /usr/bin/docker
               type: File
               resources:
                 limits:
                   cpu: 1500m
                   memory: 4000Mi
                 requests:
                   cpu: 1000m
                   memory: 3000Mi
           """
    }
  }
    stages {
        stage('Checkout') {
            steps {
                container('calling') {
                    script{
			def vars = checkout scm
			vars.each { k,v -> env.setProperty(k, v) }
			gitHash = (sh (returnStdout: true, script: 'git log -1 --pretty=%h'))
                        gitHash=gitHash.trim() }
                    sh 'git log -1 --pretty=%h'
                    sh 'ls -l'
                }
                sh 'java -version'
            }
        }
        
        stage('Create image') {
            steps {
                container('calling') {
                    sh "/usr/bin/docker build -t $IMAGE_NAME:$IMAGE_BUILD_TAG."+gitHash+" -t $IMAGE_NAME:$IMAGE_TAG -f Dockerfile ."
                    withCredentials([file(credentialsId: 'jenkins-gcr-publisher', variable: 'KEY_FILE')]) {
                        sh 'docker login -u _json_key  -p "$(cat $KEY_FILE)" https://eu.gcr.io'
                        sh "docker push $IMAGE_NAME:$IMAGE_BUILD_TAG."+gitHash
                        sh "docker push $IMAGE_NAME:$IMAGE_TAG"
                    }
                }
            }
        }
    }
}
