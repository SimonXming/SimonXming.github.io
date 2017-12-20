---
layout: post
title: jenkins 声明式(declarative) pipeline
category: 工具
tags: jenkins pipeline 声明式编程
keywords: pipeline
description:
---

# Jenkins Declarative Pipeline Example

## Basic-0x01

```groovy
pipeline {
    agent {
        node { label 'some-node' } 
    }

    parameters {
        string(name: 'TAG', defaultValue: 'v2.0.0', description: '生成镜像 tag')
        string(name: 'IMAGE_NAME', defaultValue: 'dcos-config-center', description: '生成镜像名字')
        string(name: 'DOCKERFILE', defaultValue: "Dockerfile.alpine", description: 'docker 构建 dockerfile 路径')
        string(name: 'DOCKER_CONTEXT', defaultValue: '.', description: 'docker 构建 PATH')
    }

    stages {
        stage('Preprocess') {
            steps { 
                sh """
                    env
                    cat Jenkinsfile
                    sed "s/FROM *\\(.*\\)\\/\\(.*\\):\\(.*\\)/FROM $REGISTRY_URI\\/\\2:\\3/g" Dockerfile.tmplt.alpine > Dockerfile.alpine
                    ls
                    git rev-parse HEAD > gitversion
                    date "+%Y-%m-%d %H:%M:%S" >> gitversion
                """
            }
        }
        stage('Docker Build'){
            steps {
                // Build and push image with Jenkins' docker-plugin
                withDockerRegistry(registry: [url: "http://${env.REGISTRY_URI}", credentialsId: null]) {
                    withDockerServer(server: [uri: "unix:///var/run/docker.sock", credentialsId: null]) {
                        script {
                            def image
                            image = docker.build("${env.REGISTRY_URI}/${params.IMAGE_NAME}:${params.TAG}", "-f ${params.DOCKERFILE} ${params.DOCKER_CONTEXT}")
                        }
                    }
                }
            }
        }
        stage('Deploy') {
            steps {
                sh 'echo deploy'
            }
        }
    }
}
```

## Scripted Pipeline-0x02
```groovy
node('nfs-docker') {
    stage('clean workspace') {
        cleanWs()
        echo 'clean workspace success !'
    }
    stage('install git') {
        sh '''
        if hash git 2>/dev/null; then
            echo "Git already installed!"
            exit 0
        fi
        apt-get update
        apt-get install -y build-essential
        apt-get install -y git
        '''
    }
    stage('checkout scm') {
        def branches = "*/" + params.BRANCH
        checkout([
            $class: 'GitSCM',
            branches: [[name: branches]],
            doGenerateSubmoduleConfigurations: false,
            extensions: [[
              $class: 'SubmoduleOption',
              disableSubmodules: false,
              parentCredentials: true,
              recursiveSubmodules: true,
              reference: '',
              trackingSubmodules: false
            ]],
            submoduleCfg: [],
            userRemoteConfigs: [
                [credentialsId: 'ssh-auth', url: 'git@gitlab.example.com:devtools/namespace/repo.git']
            ]])
    }
    stage('install docker') {
        sh '''
        if hash docker 2>/dev/null; then
            echo "Docker already installed!"
            exit 0
        fi

        curl -fsSL get.docker.com -o get-docker.sh
        apt-get -y -f install
        sh get-docker.sh
        '''
    }
    stage('build docker') {
        def customImage = docker.build("some-image:latest")
    }

}
```

参考阅读：
* [Jenkins pipeline syntax](https://jenkins.io/doc/book/pipeline/syntax/)  官方文档
* [Jenkins pipeline steps](https://jenkins.io/doc/pipeline/steps/) 官方文档
* [Declarative Programming](https://en.wikipedia.org/wiki/Declarative_programming)
