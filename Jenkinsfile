pipeline {
  agent any
  stages {
    stage('pull rules') {
      steps {
        git(url: 'https://github.com/tidbcloud/runbooks.git', branch: 'NewRuleFolder', poll: true)
      }
    }

  }
}