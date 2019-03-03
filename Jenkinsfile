/*
 * Requires: https://github.com/RedHatInsights/insights-pipeline-lib
 */

@Library("github.com/RedHatInsights/insights-pipeline-lib") _

node {
    cancelPriorBuilds()

    runIfMasterOrPullReq {
        runStages()
    }
}

def runStages() {

    def label = "test-${UUID.randomUUID().toString()}"

    podTemplate(
        cloud: "cmqe",
        name: "compliance-backend-test",
        label: label,
        inheritFrom: "compliance-backend-test",
        serviceAccount: pipelineVars.jenkinsSvcAccount
    ) {
        node(label) {

            checkout scm

            stage("Bundle install") {
                runBundleInstall()
            }

            stage("Prepare the db") {
                withStatusContext.dbMigrate {
                    sh "bundle exec rake db:migrate --trace"
                    sh "bundle exec rake db:test:prepare"
                }
            }

            stage("Unit tests") {
                withCredentials([string(credentialsId: "codecov_token", variable: "CODECOV_TOKEN")]) {
                    withStatusContext.unitTest {
                        sh "bundle exec rake test:validate"
                    }
                }
            }
        }
    }

    scmVars = checkout scm

    if (currentBuild.currentResult == "SUCCESS" && env.BRANCH_NAME == "master") {

        changedFiles = changedFiles()

        if ("Gemfile.lock" in changedFiles || "Gemfile" in changedFiles || "openshift/Jenkins/Dockerfile" in changedFiles) {
            // If Gemfiles or Jenknis slave's Dockerfile changed we need to rebuild the jenkins slave image
            stage("Rebuild the jenkins slave ruby image") {
                openshift.withCluster("cmqe") {
                    openshift.startBuild("jenkins-slave-base-centos7-ruby25-openscap")
                }
            }
        }


        openshift.withCluster("dev_cluster") {
            openshift.withCredentials("compliance-token") {
                openshift.withProject("compliance-ci") {
                    stage("Wait until deployed") {
                        parallel(
                            "API": {
                                timeout(20) {
                                    waitUntil {
                                        def pod = openshift.selector("pod", [name : "compliance-backend"]).name()
                                        echo "API pod: ${pod}" 
                                        try {
                                            def commitHash = openshift.rsh("${pod} git rev-parse HEAD").out
                                            echo "API pod git commit: ${commitHash}"
                                            echo "master hash ${scmVars.GIT_COMMIT}"
                                            return (commitHash == scmVars.GIT_COMMIT)
                                        } catch(e) {
                                            return false
                                        }
                                    }
                                }
                            },
                            "Consumer": {
                                timeout(20) {
                                    waitUntil {
                                        def pod = openshift.selector("pod", [name : "compliance-consumer"]).name()
                                        echo "Consumer pod: ${pod}" 
                                        try {
                                            def commitHash = openshift.rsh("${pod} git rev-parse HEAD").out
                                            echo "Consumer pod git commit: ${commitHash}"
                                            return (commitHash == scmVars.GIT_COMMIT)
                                        } catch(e) {
                                            return false
                                        }
                                    }
                                }
                            }
                        )
                    }
                }
            }
        }


        openShift.withNode(cloud: "cmqe", image: pipelineVars.jenkinsSlaveIqeImage) {
            stage("Install integration tests env") {
                sh "iqe plugin install iqe-compliance-plugin"
                sh "iqe plugin install iqe-red-hat-internal-envs-plugin"
            }

            stage("Inject credentials and settings") {
                withCredentials([
                    file(credentialsId: "compliance-settings-credentials-yaml", variable: "creds"),
                    file(credentialsId: "compliance-settings-local-yaml", variable: "settings")]
                ) {
                    sh "cp \$creds \$IQE_VENV/lib/python3.6/site-packages/iqe_compliance/conf"
                    sh "cp \$settings \$IQE_VENV/lib/python3.6/site-packages/iqe_compliance/conf"
                }
            }

            stage("Run integration tests") {
                withStatusContext.integrationTest {
                    withEnv(["ENV_FOR_DYNACONF=ci"]) {
                       sh "iqe tests plugin compliance -v -s -k 'test_validate_compliance_report or test_graphql_smoke'"    
                    }
                }
            }
        }
    }
}
