library 'global-shared-library'
import groovy.json.JsonSlurperClassic

def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}
def global_config = ""
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
                        println("CAUSE: " + cause._class.toString())
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
                    workspace_path = "${WORKSPACE}"
                    active_choice_params = input message: "Please, provide additional parameters:",
                    ok: "Run",
                    parameters: [
                        string(name: 'chart_name', defaultValue: '', description: 'Name of Helm chart to deploy'),
                        [
                          $class: 'CascadeChoiceParameter',
                          choiceType: 'PT_SINGLE_SELECT',
                          name: 'chart_path',
                          description: 'Select environment',
                          filterLength: 1,
                          filterable: true,
                          script:
                            [
                                $class: 'GroovyScript',
                                fallbackScript:
                                    [
                                        sandbox: true,
                                        script: 'return ["ERROR"]'
                                    ],
                                script:
                                    [
                                        sandbox: true,
                                        script: """
                                            try {
                                                command = "cd ${workspace_path} && echo \\\$(ls -d */) | sed 's|/||g' | tr -d '\\\n'"
                                                process = ["/bin/bash", "-c", command].execute()
                                                def standardOut = new StringBuffer()
                                                def standardErr = new StringBuffer()
                                                process.consumeProcessOutput(standardOut, standardErr)
                                                process.waitFor()
                                                if (standardOut.size() > 0){
                                                    return standardOut.tokenize()
                                                } else if (standardErr.size() > 0){
                                                    return [standardErr.toString().trim()]
                                                }
                                            } catch (error){
                                                return [error.toString()]
                                            }
                                        """
                                    ]
                            ]
                       ],
                       string(name: 'chart_values', defaultValue: '', description: 'Values in format image.repo=value1,image.tag=value2; Leave empty if not needed'),
                       string(name: 'values_file', defaultValue: '', description: 'Path to values file inside of Helm chart; Leave empty if not needed'),
                       string(name: 'cluster_name', defaultValue: '', description: 'GKE cluster name'),
                       string(name: 'region', defaultValue: '', description: 'Region where GKE is deployed'),
                       string(name: 'project', defaultValue: '', description: 'GCP project where GKE is deployed')
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
                        chart_path: active_choice_params["chart_path"],
                        values: values,
                        values_file: active_choice_params["values_file"]
                    ]
                    gke_args = [
                        cluster_name: active_choice_params["cluster_name"],
                        region: active_choice_params["region"],
                        project: active_choice_params["project"]
                    ]
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
