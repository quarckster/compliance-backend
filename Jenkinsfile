pipeline {
  agent any
  stages {
    stage('build') {
      steps {
        openshiftBuild 'sgrsr'
      }
    }
  }
}