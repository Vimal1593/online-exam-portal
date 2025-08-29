pipeline {
  agent { label 'slave_node1' }
  stages {
    stage('Checkout') {
      steps {
        // Multibranch-friendly: uses the same repo/branch/credentials as the job
        deleteDir()          // clean workspace
        checkout scm         // checkout the current branch
        sh 'ls -la'          // optional: verify files are present
      }
    }
  }
}
