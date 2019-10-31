node('build-scaleway-x64-ubuntu-16-04-2') {
  try {
    stage('Preparation') {
      properties([buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')), [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false], [$class: 'JiraProjectProperty'], pipelineTriggers([pollSCM('@hourly')])])
      checkout changelog: false, scm: [$class: 'MercurialSCM', credentialsId: '', installation: '(Default)', source: 'https://hg.openjdk.java.net/jmc/jmc']
      fileOperations([fileCreateOperation(fileContent: '''<settings>
         <profiles>
           <profile>
             <id>jmc</id>
             <repositories>
               <repository>
                 <id>jmc-publish</id>
                 <snapshots>
                   <enabled>false</enabled>
                 </snapshots>
                 <url>https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-libs</url>
                 <layout>default</layout>
               </repository>
               <repository>
                 <id>jmc-publish-snapshot</id>
                 <releases>
                   <enabled>false</enabled>
                 </releases>
                 <url>https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-libs-snapshots</url>
                 <layout>default</layout>
               </repository>
             </repositories>
           </profile>
         </profiles>
         <servers>
           <server>
             <id>jmc-publish</id>
             <username>${publish.user}</username>
             <password>${publish.password}</password>
           </server>
           <server>
             <id>jmc-publish-snapshot</id>
             <username>${publish.user}</username>
             <password>${publish.password}</password>
           </server>
         </servers>
         <activeProfiles>
           <activeProfile>jmc</activeProfile>
         </activeProfiles>
       </settings>''', fileName: '.m2/settings.xml')])
    }
    dir('workspace') {
      deleteDir()
      git 'https://github.com/reinhapa/openjdk-jmc-overrides.git'
    }
    withEnv(["JAVA_HOME=${tool 'JDK8 u172'}", "PATH=$PATH:${tool 'apache-maven-3.5.3'}/bin"]) {
      dir('core') {
        stage('Build & test core libraries') {
          // Run the maven build
          sh 'mvn clean'
//          sh 'mvn install'
          sh 'mvn install -DskipTests'
        }
        stage('Deploy core libraries') {
          withCredentials([usernamePassword(credentialsId: 'missioncontrol-jenkins-bot', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            sh "mvn deploy --settings ${WORKSPACE}/.m2/settings.xml -Dpublish.user=${USERNAME} -Dpublish.password=${PASSWORD} -Drelease.repo=https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-libs -Dsnapshot.repo=https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-libs-snapshots -Dgpg.skip=true -DskipTests=true"
          }
        }
      }
      stage('Build') {
        // Run the maven build
        dir('releng/third-party') {
          sh 'mvn clean'
          sh 'mvn p2:site'
          sh 'mvn jetty:run &'
        }
        sh 'cp workspace/overrides/latest . -rf'
        sh 'mvn package'
      }
      wrap([$class: 'Xvfb', additionalOptions: '', assignedLabels: '', autoDisplayName: true, displayNameOffset: 0, installationName: 'default', screen: '']) {
        stage('Unit Tests') {
          echo 'currently disabled'
//          sh 'mvn verify'
        }
        stage('UI Tests') {
          echo 'currently disabled'
//          sh 'mvn verify -P uitests'
        }
      }
      stage('Deploy update sites') {
        withCredentials([usernamePassword(credentialsId: 'missioncontrol-jenkins-bot', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
          sh 'unzip application/org.openjdk.jmc.updatesite.ide/target/*.zip -d workspace/repository'
          dir('workspace/repository') {
            sh 'curl -X DELETE -u "${USERNAME}:${PASSWORD}" https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-snapshots/ide'
            sh 'find . -type f -exec curl -o /dev/null -s -u "${USERNAME}:${PASSWORD}" -T \'{}\' https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-snapshots/ide/\'{}\' \\;'
          }
        }
      }
      stage('Archive artifacts') {
        junit '**/target/surefire-reports/TEST-*.xml'
        archiveArtifacts 'target/products/*'
        archiveArtifacts 'application/org.openjdk.jmc.updatesite.ide/target/*.zip'
      }
    }
  } finally {
    stage('Test results') {
      junit '**/target/surefire-reports/TEST-*.xml'
    }
  }
}
