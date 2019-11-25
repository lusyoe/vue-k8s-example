#!/usr/bin/env groovy

def projectProperties = [
        [$class: 'BuildDiscarderProperty', strategy: [$class: 'LogRotator', numToKeepStr: '5']],
        parameters([
            string(name: 'DOCKER_USER', defaultValue: '', description: 'docker用户名'),
            string(name: 'DOCKER_PASSWORD', defaultValue: '', description: 'docker用户密码'),
            string(name: 'REGISTRY_URL', defaultValue: 'docker.io', description: 'docker仓库地址')
        ])
]

properties(projectProperties)


podTemplate(cloud: 'kubernetes', containers: [
        containerTemplate(name: 'nodejs', image: 'node:10-alpine', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl:v1.14.0', command: 'cat', ttyEnabled: true)
        ],
        volumes: [
                    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
                    hostPathVolume(hostPath: '/root/.kube', mountPath: '/root/.kube'),
                    hostPathVolume(hostPath: '/etc/localtime', mountPath: '/etc/localtime')
            ]
) {
    // POD_LABEL是kubernetes插件1.17.0版本之后的一个新特性
    // 将会根据流水线名称和构建序号自动生成
    node(POD_LABEL) {

        def imgTag = sh(script: "date '+%Y%m%d%H%M%S'", returnStdout: true).trim()

        stage('checkout') {
            checkout scm

            sh 'printenv'
        }

        stage('compile') {
            container('nodejs') {
                sh "npm config set registry https://registry.npm.taobao.org"
                sh 'npm install'
                sh 'npm run build'
            }
        }

        stage('docker-build') {
            container('docker') {
                //REGISTRY_URL私有仓库地址，也可使用官方地址：docker.io
                sh """
                  docker login -u ${params.DOCKER_USER} -p ${params.DOCKER_PASSWORD} ${params.REGISTRY_URL}
                  docker build . -t ${params.REGISTRY_URL}/lusyoe/vue-example:${imgTag}
                  docker push ${params.REGISTRY_URL}/lusyoe/vue-example:${imgTag}
                  docker rmi ${params.REGISTRY_URL}/lusyoe/vue-example:${imgTag}
                  """
            }
        }

        stage('k8s deploy') {
            container('kubectl') {
                sh "sed -i \"s/lusyoe\\/vue-example/${params.REGISTRY_URL}\\/lusyoe\\/vue-example:${imgTag}/g\" k8s-example.yaml"
                sh "kubectl --kubeconfig=/root/.kube/config apply -f k8s-example.yaml"
            }
        }

    }
}

// vim: ft=groovy
