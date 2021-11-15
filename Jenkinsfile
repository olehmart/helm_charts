import groovy.json.JsonSlurperClassic

def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}
def global_config = ""
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
        choice(name: 'helm_chart', choices: ['java-hello-world', 'init'], description: 'Choose helm chart'),
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
        stage("Gather additional parameters") {
            steps {
                script {
                    configFileProvider(
                        [configFile(fileId: 'global_cicd_config', variable: 'GLOBAL_CONFIG')]) {
                        sh "cat ${GLOBAL_CONFIG}"
                        global_config = jsonParse(new File(GLOBAL_CONFIG).text)["helm_charts"]
                    }
                }
            }
        }
        stage("Get GKE credentials") {
            steps {
                sh "gcloud container clusters get-credentials ${global_config[params.helm_chart]["environments"][environment]["gke_cluster"]["name"]} --region ${global_config[params.helm_chart]["environments"][environment]["gke_cluster"]["region"]} --project ${global_config["common"]["environments"][environment]["project_id"]}"
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
                script {
                    if (params.environment != "dev") {
                        input message: 'Deploy?', ok: 'Yes'
                    }
                    sh "helm upgrade --wait --install ${params.helm_chart} ${params.helm_chart} --set image.tag=${params.image_tag} -f ${params.helm_chart}/${params.environment}-values.yaml"
                }
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
