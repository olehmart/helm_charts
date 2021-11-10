pipeline {
    agent {
        node {
            label 'slave'
        }
    }
    options {
        ansiColor('xterm')
    }
    stages {
        stage("Init") {
            steps {
                sh "gcloud container clusters get-credentials gke-cluster1 --region us-central1 --project peerless-robot-331021"
            }
        }
        stage("Helm validate") {
            steps {
                sh "helm lint java_hello_world/"
            }
        }
        stage("Helm deploy") {
            steps {
                sh "helm upgrade --wait --install java-hello-world java_hello_world --namespace java_hello_world"
            }
        }
    }
    post {
        always {
            script {
                cleanWs()
                sh 'echo "Workspace has been cleaned up!"'
            }
        }
    }
}
