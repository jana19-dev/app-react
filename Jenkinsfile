#!groovy

@Library('slack-notify') _

def errorMessage = "" // Used to construct error message in any stage

def isDeploymentBranch(){
  return CURRENT_BRANCH==PRODUCTION_BRANCH || CURRENT_BRANCH==DEVELOPMENT_BRANCH;
}

def getSuffix() {
  return CURRENT_BRANCH==DEVELOPMENT_BRANCH ? '-dev' : '';
}

pipeline {
  // construct global env variables
  environment {
    SITE_NAME = 'testing' // Name will be used for archive file (with suffix '-dev' if DEVELOPMENT_BRANCH)
    CHEF_RECIPE_NAME = 'app-testing' // Name of the chef recipe to trigger in Deploy stage
    PRODUCTION_BRANCH = 'master' // Source branch used for production
    DEVELOPMENT_BRANCH = 'dev' // Source branch used for development
    SLACK_CHANNEL = '#builds' // Slack channel to send build notifications
    CURRENT_BRANCH = env.GIT_BRANCH.getAt((env.GIT_BRANCH.indexOf('/')+1..-1)) // (eg) origin/master: get string after '/'
    COMMIT_MESSAGE = sh(returnStdout: true, script: 'git log -1 --pretty=%B').trim() // Auto generated
    COMMIT_AUTHOR = sh(returnStdout: true, script: 'git --no-pager show -s --format=%an').trim() // Auto generated
  }
  agent any
  stages {
    stage('Start') {
      steps {
        // Send 'Build Started' notification
        echo "Sending build started notification to slack"
        notifySlack commitMessage: COMMIT_MESSAGE, commitAuthor: COMMIT_AUTHOR
      }
    }
    stage ('Install Packages') {
      steps {
        script {
          try {
            // Install required node packages
            nodejs(nodeJSInstallationName: '10.6.0') {
              sh 'yarn 2>commandResult'
            }
          } catch (e) { if (!errorMessage) { errorMessage = "Failed while installing node packages.\n\n${readFile('commandResult').trim()}\n\n${e.message}"} }
        }
      }
    }
    stage ('Test') {
      // Skip stage if an error has occured in previous stages
      when { expression { return !errorMessage; } }
      steps {
        // Test
        script {
          try {
            nodejs(nodeJSInstallationName: '10.6.0') {
              sh 'yarn test 2>commandResult'
            }
          } catch (e) { if (!errorMessage) {errorMessage = "Failed while testing.\n\n${readFile('commandResult').trim()}\n\n${e.message}"} }
        }
      }
      post {
        always {
          // Publish junit test results
          junit testResults: 'junit.xml', allowEmptyResults: true
          // Publish clover.xml and html(if generated) test coverge report
          step([
            $class: 'CloverPublisher',
            cloverReportDir: 'coverage',
            cloverReportFileName: 'clover.xml',
            failingTarget: [methodCoverage: 75, conditionalCoverage: 75, statementCoverage: 75]
          ])
          script {
            if (!errorMessage && currentBuild.resultIsWorseOrEqualTo('UNSTABLE')) {
              errorMessage = "Insufficent Test Coverage."
            }
          }
        }
      }
    }
    stage ('Build') {
      // Skip stage if an error has occured in previous stages or if not isDeploymentBranch
      when { expression { return !errorMessage && isDeploymentBranch(); } }
      steps {
        script {
          try {
            // Build
            nodejs(nodeJSInstallationName: '10.6.0') {
              sh 'yarn build 2>commandResult'
            }
          } catch (e) { if (!errorMessage) {errorMessage = "Failed while building.\n\n${readFile('commandResult').trim()}\n\n${e.message}"} }
        }
      }
    }
    stage ('Upload Archive') {
      // Skip stage if an error has occured in previous stages or if not isDeploymentBranch
      when { expression { return !errorMessage && isDeploymentBranch(); } }
      steps {
        script {
          try {
            // Create archive
            sh 'mkdir -p ./ARCHIVE 2>commandResult'
            sh 'mv node_modules/ ARCHIVE/ 2>commandResult'
            sh 'mv build/* ARCHIVE/ 2>commandResult'
            // sh "cd ARCHIVE && tar zcf ${SITE_NAME}${getSuffix()}.tar.gz * --transform \"s,^,${SITE_NAME}${getSuffix()}/,S\" --exclude=${SITE_NAME}${getSuffix()}.tar.gz --overwrite --warning=none && cd .. 2>commandResult"
            // Upload archive to server
            // sh "scp ARCHIVE/${SITE_NAME}${getSuffix()}.tar.gz root@jana19.org:/root/ 2>commandResult"
          } catch (e) { if (!errorMessage) {errorMessage = "Failed while uploading archive.\n\n${readFile('commandResult').trim()}\n\n${e.message}"} }
        }
      }
    }
    stage ('Deploy') {
      // Skip stage if an error has occured in previous stages or if not isDeploymentBranch
      when { expression { return !errorMessage && isDeploymentBranch(); } }
      steps {
        script {
          try {
            // Deploy app
            // sh "rsync -azP ARCHIVE/ root@jana19.org:/var/www/jana19.org/"
          } catch (e) { if (!errorMessage) {errorMessage = "Failed while deploying.\n\n${readFile('commandResult').trim()}\n\n${e.message}"} }
        }
      }
    }
  }
  post {
    always {
      // Set the current build status based on errorMessage
      currentBuild.result = errorMessage == "" ? 'SUCCESS' : 'FAILURE'
      cleanWs() // Recursively clean workspace
      echo "Sending final build status notification to slack"
      notifySlack status: currentBuild.result, message: errorMessage, channel: '#builds', commitMessage: COMMIT_MESSAGE, commitAuthor: COMMIT_AUTHOR
      ]
  }
}