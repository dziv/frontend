def label = "maven-${UUID.randomUUID().toString()}"
import static groovy.io.FileType.FILES
def triggerSlackNotifications = false
def versionPrefix = "1.0"
    
podTemplate(label: label, containers: [containerTemplate(name: 'maven', image: 'docker.intuit.com/maven:3.3.9-jdk-8', ttyEnabled: true, command: 'cat')]) {
    node(label) {
        final scmVars = null
        properties([disableConcurrentBuilds()])
        echo sh(returnStdout: true, script: 'env')
        try {
          timestamps {
   				if(env.BRANCH_NAME == 'master') {
	                  stage('Checkout') {
	                    echo "Checkout stage :: Branch is master"
	                    scmVars = checkout(scm)
	                  }

                    stage('Versions Set') {
                      withCredentials([file(credentialsId: 'ibp-maven-settings-file', variable: 'IBP_MAVEN_SETTINGS_FILE')]) {
                        container('maven') {
                          sh """
                            mvn -f pom.xml -s ${IBP_MAVEN_SETTINGS_FILE} versions:set -DnewVersion=$versionPrefix.${BUILD_NUMBER}
                          """
                        }
                      }
                    }

	                  stage('Build & Deploy') {
                     	   withCredentials([usernamePassword(credentialsId: 'maven-ifdp-token', passwordVariable: 'NEXUS_RELEASE_REPO_PASSWORD', usernameVariable: 'NEXUS_RELEASE_REPO_USERNAME'), usernamePassword(credentialsId: 'maven-ifdp-token', passwordVariable: 'NEXUS_SNAPSHOT_REPO_PASSWORD', usernameVariable: 'NEXUS_SNAPSHOT_REPO_USERNAME')]) { 
	                      container('maven') {
	                      	sh """
				  echo 'deb [check-valid-until=no] http://archive.debian.org/debian jessie-backports main' > /etc/apt/sources.list.d/jessie-backports.list
  				  sed -i '/deb http:\\/\\/deb.debian.org\\/debian jessie-updates main/d' /etc/apt/sources.list
				  apt-get -o Acquire::Check-Valid-Until=false update
				  apt-get install -y --no-install-recommends openjfx
	                      	  mvn -s settings.xml clean deploy
	                      	"""
	                      }
                      }
	                  }

                    stage('Update Version') {
                        echo 'Checkout stage :: Checkout version repository from github'
                        git url: "https://github.intuit.com/tap/toolsVersionManager.git",credentialsId: 'ibp-github-creds-new-svc'
                        echo "Update Version :: Writting new version of image to toolsVersions-fds_il_tools_suite.properties"
                             sh """
                             sed -iE "s/fds_il_tools_suite_version=.*/fds_il_tools_suite_version=$versionPrefix.$BUILD_NUMBER/g" toolsVersions-fds_il_tools_suite.properties
                             cat toolsVersions-fds_il_tools_suite.properties
                             """

                         withCredentials([[$class: 'UsernamePasswordMultiBinding',credentialsId: 'ibp-github-creds-new-svc',usernameVariable: 'GIT_USERNAME',passwordVariable: 'GIT_PASSWORD']]) {
                          sh """
                              git config user.name ${GIT_USERNAME}
                              git config user.email “jenkins@jenkins.com”
                              git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.intuit.com/tap/toolsVersionManager.git
                              git add toolsVersions-fds_il_tools_suite.properties
                              git commit -m 'updating to version $versionPrefix.$BUILD_NUMBER'
                              git push --set-upstream origin master
                            """
                          }
                    }
              	} else {
                	  echo "Branch is not master :: Pre-build logic is not implemented."
             	}
          }
        } catch (e) {
               currentBuild.result = "FAILED"
               throw e
        } finally {
          if(triggerSlackNotifications) {
               //JenkinsfileUtils.notifyBuild(currentBuild.result,true, null, null, scmVars) 
          }
        }
    }
}
