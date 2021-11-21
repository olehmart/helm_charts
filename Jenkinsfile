library 'global-shared-library'
import groovy.json.JsonSlurperClassic

def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}
def global_config = ""
def global_config_infra = ""
Boolean manual_mode = false
properties ([
    parameters([
        string(name: 'image_tag', defaultValue: 'None', description: 'Provide image tag'),
        choice(name: 'environment', choices: ['dev', 'qa', 'prod'], description: 'Choose environment'),
        choice(name: 'helm_chart', choices: ['java-hello-world', 'init', 'java-app'], description: 'Choose helm chart'),
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
        stage("Init"){
            steps {
                script {
                    def causes = currentBuild.getBuildCauses()
                    for(cause in causes) {
                        cause_str = cause._class.toString()
                        if (cause_str.contains("UserIdCause") || cause_str.contains("RebuildCause")) {
                            manual_mode = true
                        }
                        else {
                            manual_mode = false
                        }
                    }
                }
            }
        }
        stage("Manual: Gather additional parameters") {
            when {
                equals expected: true, actual: manual_mode
            }
            agent {
                node {
                    label 'master'
                }
            }
            steps {
                script {
                    configFileProvider(
                        [configFile(fileId: 'global_cicd_config', variable: 'GLOBAL_CONFIG')]) {
                        global_config = jsonParse(sh(script: "cat ${GLOBAL_CONFIG}", returnStdout: true).trim())["helm_charts"]
                        clusters_list = jsonParse(sh(script: "cat ${GLOBAL_CONFIG}", returnStdout: true).trim())["infrastructure"][params.environment]["gke_clusters"]
                    }
                    cluster_array = []
                    for (cluster in clusters_list){
                        println(cluster["name"])
                        cluster_array += cluster["name"]
                    }
                    workspace_path = "${WORKSPACE}"
                    env = params.environment
                    active_choice_params = input message: "Please, provide additional parameters:",
                    ok: "Run",
                    parameters: [
                        string(name: 'chart_values', defaultValue: '', description: 'Values in format image.repo=value1,image.tag=value2; Leave empty if not needed'),
                        string(name: 'values_file', defaultValue: params.environment + "-values.yaml", description: 'Path to values file inside of Helm chart; Leave empty if not needed'),
                        string(name: 'cluster_name', defaultValue: '', description: 'GKE cluster name'),
                        string(name: 'region', defaultValue: '', description: 'Region where GKE is deployed'),
                        string(name: 'project', defaultValue: '', description: 'GCP project where GKE is deployed'),
                        choice(name: 'cluster_name_sew', choices: cluster_array, description: 'Choose environment'),
                    ]
                    Map values = [:]
                    if (active_choice_params["chart_values"] != ''){
                        active_choice_params["chart_values"] = active_choice_params["chart_values"].split(",")
                        for (value in active_choice_params["chart_values"]) {
                            values += [(value.split('=')[0]):(value.split('=')[1])]
                        }
                        values += ["image.tag": params.image_tag]
                    }
                    else {
                        values = null
                    }
                    if (active_choice_params["values_file"] == '') {
                        active_choice_params["values_file"] = null
                    }
                    helm_chart_args = [
                        chart_name: active_choice_params["chart_name"],
                        chart_path: params.helm_chart,
                        values: values,
                        values_file: active_choice_params["values_file"]
                    ]
                    gke_args = [
                        cluster_name: active_choice_params["cluster_name"],
                        region: active_choice_params["region"],
                        project: active_choice_params["project"]
                    ]
                    currentBuild.displayName = params.helm_chart + '-' + params.environment + '-' + env.BUILD_NUMBER
                    buildDescription("Chart name: ${active_choice_params["chart_name"]}")
                }
            }
        }
        stage("Auto: Gather additional parameters") {
            when {
                equals expected: false, actual: manual_mode
            }
            steps {
                script {
                    configFileProvider(
                        [configFile(fileId: 'global_cicd_config', variable: 'GLOBAL_CONFIG')]) {
                        global_config = jsonParse(sh(script: "cat ${GLOBAL_CONFIG}", returnStdout: true).trim())["helm_charts"]
                    }
                    helm_chart_args = [
                        chart_name: global_config[params.helm_chart]["chart_name"],
                        chart_path: params.helm_chart,
                        values: ["image.tag": params.image_tag],
                        values_file: global_config[params.helm_chart]["environments"][params.environment]["values_file"]
                    ]
                    gke_args = [
                        cluster_name: global_config[params.helm_chart]["environments"][params.environment]["gke_cluster"]["name"],
                        region: global_config[params.helm_chart]["environments"][params.environment]["gke_cluster"]["region"],
                        project: global_config["common"]["environments"][params.environment]["project_id"]
                    ]
                    currentBuild.displayName = params.helm_chart + '-' + params.environment + '-' + env.BUILD_NUMBER
                    buildDescription("Chart name: ${global_config[params.helm_chart]["chart_name"]}")
                }
            }
        }
        stage("Get GKE credentials") {
            steps {
                sh "gcloud container clusters get-credentials \
                ${gke_args.cluster_name} --region ${gke_args.region} --project ${gke_args.project}"
            }
        }
        stage("Helm validate") {
            steps {
                script {
                    helm.lint("${params.helm_chart}/")
                }
            }
        }
        stage("Helm dry-run") {
            steps {
                script {
                    helm.install(helm_chart_args, true)
                }
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
                    helm.install(helm_chart_args, false)
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
