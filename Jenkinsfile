#!/usr/bin/env groovy test
import groovy.transform.Field
import groovy.lang.Binding

pipeline {
    agent any
    environment {
        BRANCH_NAME = "${GIT_BRANCH.split('/').size() > 1 ? GIT_BRANCH.split('/')[1..-1].join('/') : GIT_BRANCH}"
        IMAGE_TAG = "${BRANCH_NAME}_${env.BUILD_ID}"
        registry = 'https://registry.hub.docker.com'
        repository = 'versoview/versoview-test'
        ENVIRONMENT = "dev"
        APP_NAME = "test-eduardo"
        aws_region = "ap-southeast-1"
        aws_access_key_id = credentials('aws_access_key_id')
        aws_secret_access_key = credentials('aws_secret_access_key')
    }
    triggers {
       // poll repo every 5 minute for changes !!!!
       pollSCM('*/5 * * * *')
   }
    stages {
       stage('Docker build and push') {
      
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub_generic', usernameVariable: 'username', passwordVariable: 'password')]){
                    sh 'docker login -u ${username} -p ${password}'
                    sh 'docker build . -t ${repository}:${IMAGE_TAG}'
                    sh 'docker push ${repository}:${IMAGE_TAG}'
                    sh "docker rmi ${repository}:${IMAGE_TAG}"
                    }
            }
       }

       stage('Deploy to k8s') {
            agent {
                docker {
                    image 'versoview/base-image:base_helm_6'
                    registryUrl 'https://registry.hub.docker.com'
                    args '-u root:root'
                    registryCredentialsId 'dockerhub_generic'
            }
            }
           steps {
               checkout([$class: 'GitSCM', branches: [[name: '*/main']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'GT_PAT', url: 'https://github.com/eduardo-vicentini-alphait/app.git']]])
               withCredentials([kubeconfigFile(credentialsId: "${ENVIRONMENT}_eks_kubeconfig", variable: 'KUBECONFIG')]) {
                    sh "aws configure set aws_access_key_id ${aws_access_key_id}"
                    sh "aws configure set aws_secret_access_key ${aws_secret_access_key}"
                    sh "aws configure set default.region ${aws_region}"
                    sh "mkdir ~/.kube"
                    sh "echo ${KUBECONFIG} > ~/.kube/config"
                    sh "kubectl get nodes"
                    sh "echo ${repository}:${IMAGE_TAG}"
                    sh "kubectl apply -f /app/${ENVIRONMENT}/ingress.yml -f /app/${ENVIRONMENT}/configmap.yml -f /app/${ENVIRONMENT}/secrets.yml -f /app/${ENVIRONMENT}/service.yml"
                    sh "envsubst < /app/${ENVIRONMENT}/deployment.yml | kubectl apply -f -"
                    sh "kubectl rollout status deployment ${APP_NAME}-deployment -n user-versoview-ns-${ENVIRONMENT}"
                }
           }
       }
   
    }

}
