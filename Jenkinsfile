pipeline {
    agent any
    environment {
        //be sure to replace "willbla" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "cheunglu/train-schedule"
    }
    podTemplate(containers: [
        containerTemplate(name: 'maven', image: 'maven:3.8.1-jdk-8', command: 'sleep', args: '99d'),
        containerTemplate(name: 'golang', image: 'golang:1.16.5', command: 'sleep', args: '99d')
    ]) {
        node(POD_LABEL) {
            stage('Get a Maven project') {
                git 'https://github.com/jenkinsci/kubernetes-plugin.git'
                container('maven') {
                    stage('Build a Maven project') {
                        sh 'mvn -B -ntp clean install'
                    }
                }
            }

            stage('Get a Golang project') {
                git url: 'https://github.com/hashicorp/terraform.git', branch: 'main'
                container('golang') {
                    stage('Build a Go project') {
                        sh '''
                        mkdir -p /go/src/github.com/hashicorp
                        ln -s `pwd` /go/src/github.com/hashicorp/terraform
                        cd /go/src/github.com/hashicorp/terraform && make
                        '''
                    }
                }
            }

        }
    }
    stages {
        stage('Build') {
            steps {
                echo 'Running build automation'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/trainSchedule.zip'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    app = docker.build(DOCKER_IMAGE_NAME)
                    app.inside {
                        sh 'echo Hello, World!'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'master'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'docker_hub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                input 'Deploy to Production?'
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
}