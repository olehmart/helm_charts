import groovy.json.JsonSlurperClassic

def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}

String gcr_repo = "gcr.io/peerless-robot-331021/"
def environments_info = jsonParse('''{
    "dev": {
        "project_id": "peerless-robot-331021",
        "region": "us-central1",
        "cluster_name": "gke-cluster1"
    }
}''')
properties ([
    parameters([
        string(name: 'image_tag', defaultValue: 'None', description: 'Provide image tag'),
        choice(name: 'environment', choices: ['dev', 'qa', 'prod'], description: 'Choose environment'),
        choice(name: 'helm_chart', choices: ['java-hello-world'], description: 'Choose helm chart'),
        booleanParam(name: 'dry_run', defaultValue: true, description: 'Disable to deploy helm chart')
    ])
])
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
                sh "gcloud container clusters get-credentials ${environments_info[params.environment]["cluster_name"]} --region ${environments_info[params.environment]["region"]} --project ${environments_info[params.environment]["project_id"]}"
            }
        }
        stage("Helm validate") {
            steps {
                sh "helm lint ${params.helm_chart}/"
            }
        }
        stage("Helm dry-run") {
            steps {
                sh "helm upgrade --dry-run --wait --install ${params.helm_chart} ${params.helm_chart} --set image.tag=${params.image_tag} -f ${params.helm_chart}/${params.environment}-values.yaml"
            }
        }
        stage("Helm deploy") {
            when {
                equals expected: false, actual: params.dry_run;
            }
            steps {
                input message: 'Deploy?', ok: 'Yes'
                sh "helm upgrade --wait --install ${params.helm_chart} ${params.helm_chart} --set image.tag=${params.image_tag} -f ${params.helm_chart}/${params.environment}-values.yaml"
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
