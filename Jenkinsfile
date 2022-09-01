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

def dataPlaneRulesID = '95072d4e-5676-4084-bdf1-09aca85881f2'
def controlPlaneRulesID = 'd5025c09-d861-417c-9cfe-797bd97ce0c3'

pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }
    stages {
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
                            dir("runbooks"){
                               git branch: 'NewRuleFolder', credentialsId: 'jenkins-user-for-github', url: 'https://github.com/tidbcloud/runbooks.git'
                            }
                        }
                    } catch (err) {
                        echo "Failed: ${err}"
                    }
                }
            }
        }
        stage('Read yaml file'){
            stages {
                stage('read dev rules') {
                    when {
                        changeset "test/rules/**" // "newrules/dev/tidb-cloud/**"
                    }
                    stages {
                        stage('read data-plane rules') {
                            when {
                                changeset "test/rules/data-plane/**" //"newrules/dev/tidb-cloud/data-plane/data-platform/**"
                            }
                            steps{
                                script{
                                    getDataRules(dataPath)
                                }
                            }
                        }
                        stage('read data-plane rules') {
                            when {
                                changeset "test/rules/control-plane/**" //"newrules/dev/tidb-cloud/control-plane/cloud-platform/**"
                            }
                            steps{
                                script{
                                    getControlRules(controlPath)
                                }
                            }
                        }
                    }
                }
                stage('read staging rules') {
                    steps{
                        echo 'read staging rules'
                    }
                }
                stage('read prod rules') {
                    steps{
                        echo 'read prod rules'
                    }
                }
            }
        }
        stage('Delivery'){
            stages {
                stage('Apply rules to dev') {
                    when {
                        changeset "test/rules/data-plane/**" // "newrules/dev/tidb-cloud/**"
                    }
                    stages {
                        stage('Call API to update data-plane rules') {
                            when {
                                changeset "test/rules/data-plane/**" // "newrules/dev/tidb-cloud/data-plane/data-platform/**"
                            }
                            steps {
                                script{
                                    token = getToken()
                                    writeRules("${dataPlaneRulesID}", "${dataRules}", "${globalDomainDev}", "${token}")
                                }
                            }
                        }
                        stage('Call API to update control-plane rules') {
                            when {
                                changeset "test/rules/control-plane/**" // "newrules/dev/tidb-cloud/control-plane/cloud-platform/**"
                            }
                            steps {
                                script{
                                    token = getToken()
                                    writeRules("${controlPlaneRulesID}", "${controlRules}", "${globalDomainDev}", "${token}")
                                }
                            }
                        }
                    }
                }
                stage('Apply rules to staging') {
                    steps{
                        echo 'Apply rules to staging'
                    }
                }
                stage('Apply rules to prod') {
                   steps{
                        echo 'Apply rules to prod'
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

