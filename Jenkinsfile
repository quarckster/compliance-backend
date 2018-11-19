pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        openshiftBuild(bldCfg: 'compliance-backend', namespace: 'buildfactory')
      }
    }
  }
}