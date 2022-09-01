#!groovy

import groovy.transform.Field
import groovy.json.JsonOutput
import groovy.json.JsonSlurper

@Field
def dataRules = ""
@Field
def controlRules = ""

def dataPath = 'runbooks/newrules/tidb-cloud/data-plane/data-platform/'
def controlPath = 'runbooks/newrules/tidb-cloud/control-plane/cloud-platform/'
def globalDomainDev = 'k8s-observab-globalin-22824fa614-584301555.us-west-2.elb.amazonaws.com'
def globalDomainStaging = 'www.gs.us-west-2.aws.observability-staging.pingcap.cloud/'
def globalDomainProd = 'www.gs.us-west-2.aws.observability.tidbcloud.com'

pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }
    stages {
        //拉取下来存放在一个地方
        stage('Pull rules') {
            when {
                beforeAgent true
                changeRequest()
            }
            steps {
                script {
                    try {
                        retry(3) {
                            echo 'pulling rules...'
                            // 这里的路径应该是相对路径 就是这个prj之后的
                            dir("runbooks"){
                               git branch: 'NewRuleFolder', credentialsId: 'jenkins-user-for-github', url: 'https://github.com/tidbcloud/runbooks.git'
                            }
                        }
                    } catch (Exception e) {
                        println e
                    }
                }
            }
        }
        stage('Read yaml file'){
            steps{
                script{
                    try {
                        retry(3) {
                            getDataRules(dataPath)
                            getControlRules(controlPath)
                        }
                    } catch (err) {
                        echo "Failed: ${err}"
                    }
                }
            }
        }
        stage('Delivery') {
            //
            agent  any
            stage('Apply rules to dev') {
//                 environment {
//                     ENV = "dev"
//                     AWS_ROLE_ARN = credentials("dbaas-dev-aws-role") // todo: how to use
//                 }
                when {
                    beforeAgent true
                    allOf {
                        not { changeRequest() }
                    }
                    // expression {return params.CHOICES == 'dev'}
                    // directory
                    anyOf {
                        changeset "dev/**/*.yaml"
                    }
                    // branch
                    branch 'dev'
                }
                stages {
                    stage('Call API to update rules') {
                        steps {
                            script{
                                token = getToken()
                                writeRules("templateid-test-data-plane", "${dataRules}", "${globalDomainDev}", "${token}")
                                writeRules7s6("templateid-test-control-plane", "${controlRules}", "${globalDomainDev}", "${token}")
                            }
                        }

                    }
                }
            }
            stage('Apply rules to staging') {
                environment {
                    ENV = "staging"
                    AWS_ROLE_ARN = credentials("dbaas-staging-aws-role")
                }
                when {
                    beforeAgent true
                    allOf {
                        not { changeRequest() }
                    }
                    anyOf {
                        changeset "staging/**/*.yaml"
                    }
                    branch 'staging'
                }
                stages {
                    stage('Call API to update rules') {
                        steps {
                            script{
                                token = getToken()
                                writeRules("templateid-test-data-plane", "${dataRules}", "${globalDomainStaging}", "${token}")
                                writeRules("templateid-test-control-plane", "${controlRules}", "${globalDomainStaging}", "${token}")
                            }
                        }

                    }
                }
            }
            stage('Apply rules to prod') {
                environment {
                    ENV = "prod"
                    AWS_ROLE_ARN = credentials("dbaas-prod-aws-role")
                }
                when {
                    beforeAgent true
                    allOf {
                        not { changeRequest() }
                    }
                    anyOf {
                        changeset "prod/**/*.yaml"
                    }
                    branch 'prod'
                }
                stages {
                    stage('Call API to update rules') {
                        steps {
                            script{
                                token = getToken()
                                writeRules("templateid-test-data-plane", "${dataRules}", "${globalDomainProd}", "${token}")
                                writeRules("templateid-test-control-plane", "${controlRules}", "${globalDomainProd}", "${token}")
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'get rules done'
        }
        success {
            echo 'the result is success'
        }
        failure {
            echo 'the result is failure'
        }
        unstable {
            echo 'the result is unstable'
        }
        changed {
            echo 'the state of the Pipeline has changed'
        }
    }
}


def getDataRules(PATH) {
    def files
    dir(PATH) {
        files = findFiles(glob: '*.yaml')
    }
    for(file in files) {
        content = readYaml file : "${PATH}${file.path}"
        json = JsonOutput.toJson(content)
        dataRules = "${dataRules}${json.substring(11, json.length()-2)},"
    }
    dataRules = "{\"groups\":[${dataRules.substring(0, dataRules.length()-1)}]}"
}

def getControlRules(PATH) {
    def files
    dir(PATH) {
        files = findFiles(glob: '*.yaml')
    }
    for(file in files) {
        content = readYaml file : "${PATH}${file.path}"
        json = JsonOutput.toJson(content)
        controlRules = "${controlRules}${json.substring(11, json.length()-2)},"
    }
    controlRules = "{\"groups\":[${controlRules.substring(0, controlRules.length()-1)}]}"
}

def getToken() {
    def response = sh returnStdout: true, script: """
    curl --location --request POST 'https://tidb-soc2.us.auth0.com/oauth/token' \
    --header 'content-type: application/x-www-form-urlencoded' \
    --header 'Cookie: did=s%3Av0%3A0084f670-2208-11ed-b5a3-734d8378f485.gHmjM9dSvukwVpPO0K7JerA3DuKat3Jp4QjL3VQVYaE; did_compat=s%3Av0%3A0084f670-2208-11ed-b5a3-734d8378f485.gHmjM9dSvukwVpPO0K7JerA3DuKat3Jp4QjL3VQVYaE' \
    --data-urlencode 'grant_type=client_credentials' \
    --data-urlencode 'client_id=Sd4aekPCa5nMUSQBIvf8eGSL895WtV4o' \
    --data-urlencode 'client_secret=-SLP5UvxJdwGbVaKupxgv53T6otXguApPoG0fBsWCIqDG8_TuvlYnlkhPQHcWaKT' \
    --data-urlencode 'audience=https://tidb-soc2.us.auth0.com/api/v2/'
    """
    def jsonSlurper = new JsonSlurper()
    def object = jsonSlurper.parseText(response)

    echo "${object}"
    echo object.access_token
    return object.access_token
}


def writeRules(RULES_ID, RULES, DOMAIN, TOKEN) {
    sh """
    curl --location --request POST 'http://${DOMAIN}/api/v1/alertrule-template' \
    --header 'Authorization: Bearer ${TOKEN}' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "vendor": "aws",
        "region": "us-west-2",
        "templateID": "${RULES_ID}",
        "monitorPromRules": ${RULES}
    }'
    """
}

