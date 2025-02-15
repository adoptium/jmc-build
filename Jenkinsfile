def overridesUrl = 'https://github.com/adoptium/jmc-build.git'
def overridesBranch = 'main'
def jmcBranch = 'master'
def jmcVersion = '9.1.0-SNAPSHOT'

pipeline {
    agent {
        // The ubuntu pod template allows UI tests.
        kubernetes {
            inheritFrom 'ubuntu-2404'
        }
    }
    tools {
        maven 'apache-maven-3.9.9'
        jdk 'temurin-jdk21-latest'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
//        pipelineTriggers([cron('H 20 * * *')])
    }

    environment {
        MAVEN_OPTS = '-Xmx2048m -Declipse.p2.mirrors=false'
    }

    stages {
        stage('Preparation') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: jmcBranch]],
                    doGenerateSubmoduleConfigurations: false,
                    extensions: [],
                    submoduleCfg: [],
                    userRemoteConfigs: [[url: 'https://github.com/openjdk/jmc.git']]
                ])
                dir('workspace') {
                    git branch: overridesBranch, url: overridesUrl
                }
                // apply overrides
                sh 'cp workspace/overrides/* . -rvf'
                sh 'mkdir .m2 && cp workspace/.github/workflows/settings.xml .m2/'
            }
        }

        stage('Build & test core libraries') {
            steps {
                dir('core') {
                    // Run the maven build
                    sh 'mvn install'
                }
            }
        }

        stage('Build & test agent') {
            steps {
                dir('agent') {
                    // Run the maven build
                    sh 'mvn install'
                }
            }
        }

        stage('Build') {
            steps {
                // Run the maven build
                dir('releng/third-party') {
                    sh 'mvn clean'
                    sh 'mvn p2:site'
                    sh 'mvn jetty:run &'
                }
                sh 'mvn package'
            }
        }

        stage('Tests') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    timeout(180) {
                        xvnc {
                            sh 'mvn verify -fae -P uitests'
                        }
                    }
                }
            }
        }

    }

    post {
        always {
            junit '**/target/surefire-reports/TEST-*.xml'
            archiveArtifacts artifacts: 'agent/target/agent-*', fingerprint: true
            archiveArtifacts artifacts: 'application/org.openjdk.jmc.updatesite.ide/target/*.zip', fingerprint: true
            archiveArtifacts artifacts: 'target/products/org.openjdk.jmc-*', fingerprint: true
        }
        // send a mail on unsuccessful and fixed builds
        unsuccessful { // means unstable || failure || aborted
        // Email notification
            emailext subject: 'Build $BUILD_STATUS $PROJECT_NAME #$BUILD_NUMBER!',
            body: '''Check console output at $BUILD_URL to view the results.''',
            recipientProviders: [culprits(), requestor()]
            //to: 'other.recipient@domain.org'
            archiveArtifacts allowEmptyArchive: true, artifacts: '**/target/surefire-reports/TEST-*.xml', followSymlinks: false

            // Slack notification
            slackSend(
                channel: '#build-failures',
                color: 'danger',
                message: "Build *FAILED* for `${env.JOB_NAME}` - Build #${env.BUILD_NUMBER}\nSee details: ${env.BUILD_URL}"
            )
        }
        fixed { // back to normal
            // Email notification
            emailext subject: 'Build $BUILD_STATUS $PROJECT_NAME #$BUILD_NUMBER!',
            body: '''Check console output at $BUILD_URL to view the results.''',
            recipientProviders: [culprits(), requestor()]
            //to: 'other.recipient@domain.org'

            // Slack notification
            slackSend(
                channel: '#build-failures',
                color: 'good',
                message: "Build *FIXED* for `${env.JOB_NAME}` - Build #${env.BUILD_NUMBER}\nSee details: ${env.BUILD_URL}"
            )
        }
    }
}
