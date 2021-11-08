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
                sh "echo Init"
            }
        }
        stage("Helm validate") {
            steps {
                sh "echo 'Helm validate'"
            }
        }
        stage("Helm deploy") {
            steps {
                sh "echo 'Helm deploy'"
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
