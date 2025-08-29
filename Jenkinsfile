pipeline {
  agent { label 'slave_node1' }
  stages {
    stage('Test') {
      steps {
        sh 'whoami'
        sh 'java -version || true'
      }
    }
  }
}
