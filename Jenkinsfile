pipeline {
    agent any
    environment {
        //be sure to replace "surajkeshri" with your own Docker Hub username
        DOCKER_IMAGE_NAME = "kollate-218719/train-schedule"
        CANARY_REPLICAS = "0"
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
                    docker.withRegistry('https://gcr.io', 'gcr:kollate-218719') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
        stage('Canary deployment') {
            when {
                branch 'master'
            }
            environment {
                CANARY_REPLICAS = "1"
            }
            steps {
                script {
                    kubernetesDeploy(
                        kubeconfigId: 'kubeconfig',
                        configs: 'train-schedule-canary.yml',
                        enableConfigSubstitution: true
                    )
                }
            }
        }
        stage('SmokeTest') {
            when {
                branch 'master'
            }
            steps {
                script {
                    def response = httpRequest(
                        url: "http://$KUBE_MASTER_IP:30085/",
                        timeout: 30
                    )
                    if (response.status != 200) {
                        error("Smoke test agains canary failed")
                    }
                }
            }
        }
        stage('DeployToProduction') {
            when {
                branch 'master'
            }
            steps {
                milestone(1)
                kubernetesDeploy(
                    kubeconfigId: 'kubeconfig',
                    configs: 'train-schedule-kube.yml',
                    enableConfigSubstitution: true
                )
            }
        }
    }
    post {
        always {
            kubernetesDeploy(
                kubeconfigId: 'kubeconfig',
                configs: 'train-schedule-canary.yml',
                enableConfigSubstitution: true
            )
        }
    }
}
