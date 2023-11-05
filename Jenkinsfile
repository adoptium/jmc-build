def overridesUrl = 'https://github.com/adoptium/jmc-build.git'
def overridesBranch = 'master'
def jmcBranch = 'master'
def jmcVersion = '9.0.0-SNAPSHOT'

pipeline {
    agent {
        kubernetes {
            inheritFrom 'centos-8'
        }
    }
    tools {
        maven 'apache-maven-3.8.6'
        jdk 'temurin-jdk17-latest'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
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
            dir('target/products') {
              sh "mv -f org.openjdk.jmc-linux.gtk.aarch64.tar.gz    org.openjdk.jmc-${jmcVersion}-linux.gtk.aarch64.tar.gz"
              sh "mv -f org.openjdk.jmc-linux.gtk.x86_64.tar.gz     org.openjdk.jmc-${jmcVersion}-linux.gtk.x86_64.tar.gz"
              sh "mv -f org.openjdk.jmc-macosx.cocoa.aarch64.tar.gz org.openjdk.jmc-${jmcVersion}-macosx.cocoa.aarch64.tar.gz"
              sh "mv -f org.openjdk.jmc-macosx.cocoa.x86_64.tar.gz  org.openjdk.jmc-${jmcVersion}-macosx.cocoa.x86_64.tar.gz"
              sh "mv -f org.openjdk.jmc-win32.win32.x86_64.zip      org.openjdk.jmc-${jmcVersion}-win32.win32.x86_64.zip"
            }
            archiveArtifacts artifacts: 'target/products/org.openjdk.jmc-*', followSymlinks: false
        }
        // send a mail on unsuccessful and fixed builds
        unsuccessful { // means unstable || failure || aborted
            emailext subject: 'Build $BUILD_STATUS $PROJECT_NAME #$BUILD_NUMBER!',
            body: '''Check console output at $BUILD_URL to view the results.''',
            recipientProviders: [culprits(), requestor()]
            //to: 'other.recipient@domain.org'
            archiveArtifacts allowEmptyArchive: true, artifacts: '**/target/surefire-reports/TEST-*.xml', followSymlinks: false
        }
        fixed { // back to normal
            emailext subject: 'Build $BUILD_STATUS $PROJECT_NAME #$BUILD_NUMBER!',
            body: '''Check console output at $BUILD_URL to view the results.''',
            recipientProviders: [culprits(), requestor()]
            //to: 'other.recipient@domain.org'
        }
    }
}
