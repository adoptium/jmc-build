node('build-scaleway-ubuntu1604-x64-1') {
  stage('Preparation') {
    properties([
      buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '', numToKeepStr: '10')),
      [$class: 'RebuildSettings', autoRebuild: false, rebuildDisabled: false],
      [$class: 'JiraProjectProperty'], pipelineTriggers([pollSCM('@daily')])
    ])
    checkout([
      $class: 'MercurialSCM',
      credentialsId: '',
      installation: '(Default)',
      revision: '7.1.1-ga',
      revisionType: 'TAG',
      source: 'https://hg.openjdk.java.net/jmc/jmc7'
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
    git branch: '7.x.x', url: 'https://github.com/AdoptOpenJDK/openjdk-jmc-overrides.git'
  }
  // apply overrides
  sh 'cp workspace/overrides/* . -rf'
  // change to final version
  sh 'find . ! -path "*/.hg/**" -type f -name "pom.xml" -exec sed -i s/"7.1.1-SNAPSHOT"/"7.1.1"/ {} \\;'
  sh 'find . ! -path "*/.hg/**" -type f \\( -name "feature.xml" -o -name "MANIFEST.MF" \\) -exec sed -i s/"7.1.1.qualifier"/"7.1.1"/ {} \\;'
  // start build process
  withEnv(["JAVA_HOME=${tool 'JDK8 u172'}", "PATH=$PATH:${tool 'apache-maven-3.5.3'}/bin"]) {
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
    wrap([$class: 'Xvfb', additionalOptions: '', assignedLabels: '', autoDisplayName: true, displayNameOffset: 0, installationName: 'default', screen: '']) {
      stage('Unit Tests') {
        sh 'mvn verify'
      }
      stage('UI Tests') {
        echo 'currently disabled'
//        sh 'mvn verify -P uitests'
      }
    }
    stage('Deploy update sites') {
      withCredentials([usernamePassword(credentialsId: 'missioncontrol-jenkins-bot', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
// disabled until a release repo is available
//        dir('application/org.openjdk.jmc.updatesite.ide/target/repository') {
//          sh 'curl -X DELETE -u "${USERNAME}:${PASSWORD}" https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-snapshots/ide'
//          sh 'find . -type f -exec curl -o /dev/null -s -u "${USERNAME}:${PASSWORD}" -T \'{}\' https://adoptopenjdk.jfrog.io/adoptopenjdk/jmc-snapshots/ide/\'{}\' \\;'
//        }
      }
    }
    stage('Archive artifacts') {
      junit '**/target/surefire-reports/TEST-*.xml'
      dir('target/products') {
        sh 'mv -f org.openjdk.jmc-win32.win32.x86_64.zip      org.openjdk.jmc-7.1.1-win32.win32.x86_64.zip'
        sh 'mv -f org.openjdk.jmc-macosx.cocoa.x86_64.tar.gz  org.openjdk.jmc-7.1.1-macosx.cocoa.x86_64.tar.gz'
        sh 'mv -f org.openjdk.jmc-linux.gtk.x86_64.tar.gz     org.openjdk.jmc-7.1.1-linux.gtk.x86_64.tar.gz'
      }
      archiveArtifacts 'target/products/*'
      archiveArtifacts 'application/org.openjdk.jmc.updatesite.ide/target/*.zip'
    }
    stage('Test results') {
      junit '**/target/surefire-reports/TEST-*.xml'
    }
  }
}
