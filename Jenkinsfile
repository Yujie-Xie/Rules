pipeline {
  agent any
  stages {
    stage('pull rules') {
      steps {
        git(url: 'https://github.com/tidbcloud/runbooks.git', branch: 'NewRuleFolder', poll: true)
      }
    }

    stage('write rules') {
      steps {
        sh 'curl POST /api/v1/observability-regional/alertrule-template?id=${RULES_ID}&alertRules=${RULES}'
      }
    }

  }
}