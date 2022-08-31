
#!groovy

import groovy.transform.Field
import groovy.json.JsonOutput

@Field
def dataRules = ""
@Field
def controlRules = ""

def dataPath = 'newrules/tidb-cloud/data-plane/data-platform/'
def controlPath = 'newrules/tidb-cloud/control-plane/cloud-platform/'

pipeline {
    agent any
    parameters {
        choice(name:'CHOICES', choices:['dev','staging','prod'], defaultValue:"dev", description:'please choose the environment')
    }

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
//                             dir("runbooks"){
                               git branch: 'NewRuleFolder', credentialsId: 'jenkins-user-for-github', url: 'https://github.com/tidbcloud/runbooks.git'
//                             }
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
        stage("echo") {
            steps{
                script{
                    // token = getToken()
                    // writeRules("templateid-test-data-plane", "${dataRules}")
                    // writeRules("templateid-test-control-plane", "${controlRules}")

                    echo dataRules
                    echo '----------------------------'
                    echo controlRules
                }
            }
        }
        stage('Delivery') {
            agent {
                kubernetes {
                    defaultContainer "main"
                    customWorkspace "/home/jenkins/agent/workspace/go/src/github.com/tidbcloud/runbooks"
                }
            }
            stage('Apply rules to dev') {
                environment {
                    ENV = "dev" //获取当前分支或当前更新的文件
                    AWS_ROLE_ARN = credentials("dbaas-dev-aws-role") // todo: change the arn
                }
                when {
                    beforeAgent true
                    allOf {
                        not { changeRequest() }
                    }
                    expression {return params.CHOICES == 'dev'}
                }
                stages {
                    stage('Call API to update rules') {
                        steps {
                            script{
                                // token = getToken()
                                writeRules("templateid-test-data-plane", "${dataRules}")
                                writeRules("templateid-test-control-plane", "${controlRules}")
                            }
                        }

                    }
                }
            }
            stage('Apply rules to staging') {
                environment {
                    ENV = "staging" //获取当前分支或当前更新的文件
                    AWS_ROLE_ARN = credentials("dbaas-staging-aws-role") // todo: change the arn
                }
                when {
                    beforeAgent true
                    allOf {
                        not { changeRequest() }
                    }
                    expression {return params.CHOICES == 'staging'}
                }
                stages {
                    stage('Call API to update rules') {
                        steps {
                            script{
                                // token = getToken()
                                // writeRules("templateid-test-data-plane", "${dataRules}")
                                // writeRules("templateid-test-control-plane", "${controlRules}")
                            }
                        }

                    }
                }
            }
            stage('Apply rules to prod') {
                environment {
                    ENV = "prod" //获取当前分支或当前更新的文件
                    AWS_ROLE_ARN = credentials("dbaas-prod-aws-role") // todo: change the arn
                }
                when {
                    beforeAgent true
                    allOf {
                        not { changeRequest() }
                    }
                    expression {return params.CHOICES == 'prod'}
                }
                stages {
                    stage('Call API to update rules') {
                        steps {
                            script{
                                // token = getToken()
                                // writeRules("templateid-test-data-plane", "${dataRules}")
                                // writeRules("templateid-test-control-plane", "${controlRules}")
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
    --data-raw ''
    """
    echo response
    return response
}

def writeRules(RULES_ID, RULES) {
    sh """
    curl --location --request POST 'http://k8s-observab-globalin-22824fa614-584301555.us-west-2.elb.amazonaws.com/api/v1/alertrule-template' \
    --header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjZ6Zng2YUh6dlVhUTVaV3hZbVJxRiJ9.eyJpc3MiOiJodHRwczovL3RpZGItc29jMi51cy5hdXRoMC5jb20vIiwic3ViIjoiU2Q0YWVrUENhNW5NVVNRQkl2ZjhlR1NMODk1V3RWNG9AY2xpZW50cyIsImF1ZCI6Imh0dHBzOi8vdGlkYi1zb2MyLnVzLmF1dGgwLmNvbS9hcGkvdjIvIiwiaWF0IjoxNjYxOTI3MzU4LCJleHAiOjE2NjIwMTM3NTgsImF6cCI6IlNkNGFla1BDYTVuTVVTUUJJdmY4ZUdTTDg5NVd0VjRvIiwic2NvcGUiOiJyZWFkOmNsaWVudF9ncmFudHMiLCJndHkiOiJjbGllbnQtY3JlZGVudGlhbHMifQ.bVXkgPltnQpUR30b_uFRrUB4xekQOBafisij23VysG0KE39l7jmHUPu_XQtrcuJvcLD9RqOuqBeeDNMv50l6gFwJVe_5m6vBYxKaFNkcawOz6UsDai7R1043BmtEuatCLDf4QmshJU8ALnQwv1dAQWnWyOVZkfpgRaJ3dZamoTbZswkLAPp31bx9ELZLb_f2EhA2l4KeUeUNtOkspdpDiacIegKA8hWahwX1skf7xxAXus7AS46KK_P6D_3LjhaLtf-BylGg0t5IFUXCPhWaPXLX134Sy3nDIR3ZF3RFJedOd08wg3igy1WmCnAjvJ9PUS5D5Ucsi1VlwlyJks2LvA' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "vendor": "aws",
        "region": "us-west-2",
        "templateID": "${RULES_ID}",
        "monitorPromRules": ${RULES}
    }'
    """
}



