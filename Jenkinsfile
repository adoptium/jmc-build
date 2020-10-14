node('build-scaleway-ubuntu1604-x64-1') {
  stage('Preparation') {
    properties([
      buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')), 
      [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false], 
      [$class: 'JiraProjectProperty'], pipelineTriggers([pollSCM('@daily')])
    ])
    checkout([
      $class: 'GitSCM', 
      branches: [[name: '*/master']], 
      doGenerateSubmoduleConfigurations: false, 
      extensions: [], 
      submoduleCfg: [], 
      userRemoteConfigs: [[url: 'https://github.com/openjdk/jmc.git']]
    ])
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
    git 'https://github.com/AdoptOpenJDK/openjdk-jmc-overrides.git'
  }
  // apply overrides
  sh 'cp workspace/overrides/latest/* . -rf'
  // start build process
  withEnv(["JAVA_HOME=${tool 'JDK11'}", "PATH=$PATH:${tool 'apache-maven-3.5.3'}/bin"]) {
  	// print some info about the used JDK
    sh '$JAVA_HOME/bin/java -version'
    dir('core') {
      stage('Build & test core libraries') {
        // Run the maven build
        sh 'mvn clean'
        sh 'mvn install'
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
      sh 'mvn package'
    }
    try {
      wrap([$class: 'Xvfb', additionalOptions: '', assignedLabels: '', autoDisplayName: true, displayNameOffset: 0, installationName: 'default', screen: '']) {
        stage('Unit Tests') {
          sh 'mvn verify'
        }
        stage('UI Tests') {
          echo 'currently disabled'
          try {
            sh 'mvn verify -P uitests'
          } catch (e) {
            echo  'ignoring error for now'
          }
        }
      }
    } finally {
      stage('Collect test results') {
        junit '**/target/surefire-reports/TEST-*.xml'
      }
    }
    stage('Deploy update sites') {
      withCredentials([usernamePassword(credentialsId: 'missioncontrol-jenkins-bot', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
        dir('application/org.openjdk.jmc.updatesite.ide/target/repository') {
          sh 'curl -X DELETE -u "${USERNAME}:${PASSWORD}" https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-snapshots/ide'
          sh 'find . -type f -exec curl -o /dev/null -s -u "${USERNAME}:${PASSWORD}" -T \'{}\' https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-snapshots/ide/\'{}\' \\;'
        }
      }
    }
    stage('Archive artifacts') {
      junit allowEmptyResults: true, testResults: '**/target/surefire-reports/TEST-*.xml'
      dir('target/products') {
        sh 'mv -f org.openjdk.jmc-win32.win32.x86_64.zip      org.openjdk.jmc-8.0.0-SNAPSHOT-win32.win32.x86_64.zip'
        sh 'mv -f org.openjdk.jmc-macosx.cocoa.x86_64.tar.gz  org.openjdk.jmc-8.0.0-SNAPSHOT-macosx.cocoa.x86_64.tar.gz'
        sh 'mv -f org.openjdk.jmc-linux.gtk.x86_64.tar.gz     org.openjdk.jmc-8.0.0-SNAPSHOT-linux.gtk.x86_64.tar.gz'
      }
      archiveArtifacts 'target/products/*'
      archiveArtifacts 'application/org.openjdk.jmc.updatesite.ide/target/*.zip'
    }
  }
}
