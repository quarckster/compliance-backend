/*
 * Requires: https://github.com/RedHatInsights/insights-pipeline-lib
 */

@Library("github.com/RedHatInsights/insights-pipeline-lib") _

// Code coverage failure threshold
// codecovThreshold = 0


node {
    cancelPriorBuilds()

    runIfMasterOrPullReq {
        runStages()
    }
}


def runStages() {
    openShift.withNode(image: "docker-registry.default.svc:5000/jenkins/jenkins-slave-base-centos7-ruby25-openscap:latest") {
        
        environment {
            DATABASE_SERVICE_NAME = "compliance-db"
            COMPLIANCE_DB_SERVICE_NAME = "compliance-db.compliance-ci.svc"
            POSTGRESQL_USER = "compliance_user"
            POSTGRESQL_PASSWORD = "compliance_password"
            POSTGRESQL_ADMIN_PASSWORD = "db_admin_password"
            POSTGRESQL_DATABASE = "compliance"
            POSTGRESQL_MAX_CONNECTIONS = "100"
            POSTGRESQL_SHARED_BUFFERS = "12MB"
            RAILS_ENV = "test"
        }

        stage('Lint') {
            sh "bundle exec rake validate"
        }

        stage('Install gems') {
            sh "bundle install --path ./bundle"
        }

        stage('Prepare the db') {
            steps {
                sh "bundle exec rake db:migrate --trace"
                sh "bundle exec rake db:test:prepare"
            }
        }

        stage('UnitTest') {
            withStatusContext.unitTest {
                sh "bundle exec rake test"
            }
        }

        if (currentBuild.currentResult == 'SUCCESS') {
            if (env.BRANCH_NAME == 'master') {
                // Stages to run specifically if master branch was updated
            }
        }
    }
}
