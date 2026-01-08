pipeline {
  agent any

  parameters {
    string(
      name: 'GIT_REPO_URL',
      defaultValue: 'https://github.com/Vimal1593/online-exam-portal.git',
      description: 'Git repository URL'
    )
    string(
      name: 'GIT_BRANCH',
      defaultValue: 'testa',
      description: 'Git branch to checkout'
    )
    string(
      name: 'BUILD_SRC',
      defaultValue: '/var/www/build/abstract/payables/backed',
      description: 'Build artifact source directory'
    )
    string(
      name: 'APP_BASE_DIR',
      defaultValue: '/var/www/application/payables/api',
      description: 'Target application directory'
    )
    string(
      name: 'SECURITY_EMAIL',
      defaultValue: 'security@company.com',
      description: 'Email to send Snyk approval request'
    )
  }

  environment {
    SNYK_BLOCKED = "false"
  }

  options {
    timestamps()
  }

  stages {

    /* =========================
       STAGE 0: GIT CHECKOUT
       ========================= */
    stage('Git Checkout') {
      steps {
        echo "Checking out source code..."
        checkout([
          $class: 'GitSCM',
          branches: [[name: "*/${params.GIT_BRANCH}"]],
          userRemoteConfigs: [[
            url: params.GIT_REPO_URL
            // credentialsId: 'GIT_CREDENTIALS_ID' // uncomment if private repo
          ]]
        ])
      }
    }

    /* =========================
       STAGE 1: COPY BUILD FILES
       ========================= */
    stage('Prepare Application') {
      steps {
        sh '''
          echo "Copying build artifacts..."
          echo "Source : ${BUILD_SRC}"
          echo "Target : ${APP_BASE_DIR}"

          mkdir -p ${APP_BASE_DIR}
          cp -r ${BUILD_SRC}/* ${APP_BASE_DIR}/
        '''
      }
    }

    /* =========================
       STAGE 2: SNYK AUTH + BASE SETUP
       ========================= */
    stage('Snyk Auth & Base Dependencies') {
      steps {
        withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
          sh '''
            cd ${APP_BASE_DIR}
            snyk auth ${SNYK_TOKEN}
            npm install dotenv
          '''
        }
      }
    }

    /* =========================
       STAGE 3: DASHBOARD SERVICE INSTALL
       ========================= */
    stage('Dashboard Service Install') {
      steps {
        sh '''
          cd ${APP_BASE_DIR}/dashboard-service
          npm install
        '''
      }
    }

    /* =========================
       STAGE 4: SNYK SCAN (SCA + SAST)
       ========================= */
    stage('Snyk Security Scan') {
      steps {
        script {
          sh '''
            set +e
            cd ${APP_BASE_DIR}
            python3 snyk_report.py .
            echo $? > scan_rc.txt
          '''

          def rc = readFile("${params.APP_BASE_DIR}/scan_rc.txt").trim()

          if (rc != "0") {
            echo "High / Critical vulnerabilities detected"
            env.SNYK_BLOCKED = "true"
            currentBuild.result = "UNSTABLE"
          } else {
            echo "No High / Critical vulnerabilities found"
          }
        }
      }
    }

    /* =========================
       STAGE 5: SECURITY APPROVAL
       ========================= */
    stage('Security Approval') {
      when {
        expression { env.SNYK_BLOCKED == "true" }
      }
      steps {
        script {

          emailext(
            to: params.SECURITY_EMAIL,
            subject: "Approval Required: Snyk High/Critical Vulnerabilities",
            body: """
High / Critical vulnerabilities were detected during the Snyk scan.

Git Repo : ${params.GIT_REPO_URL}
Branch   : ${params.GIT_BRANCH}
App Path : ${params.APP_BASE_DIR}

Job      : ${env.JOB_NAME}
Build    : ${env.BUILD_NUMBER}

Attached:
- Snyk Security Report (PDF)

Actions:
✔ Approve → Click Approve button
✖ Deny    → Click Abort / Cancel

Approval Link:
${env.BUILD_URL}input/
""",
            attachmentsPattern: "${params.APP_BASE_DIR}/snyk-results/*.pdf"
          )

          try {
            timeout(time: 30, unit: 'MINUTES') {
              input(
                message: 'High/Critical vulnerabilities found. Approve risk to continue?',
                ok: 'Approve Deployment',
                submitterParameter: 'APPROVED_BY'
              )
            }

            echo "Security risk approved by: ${APPROVED_BY}"

          } catch (err) {

            sh '''
              echo "=================================================="
              echo "SECURITY APPROVAL DENIED"
              echo "The user did not accept the security risk."
              echo "Deployment is blocked as per security policy."
              echo "=================================================="
            '''

            error("Build stopped: Security approval denied.")
          }
        }
      }
    }

    /* =========================
       STAGE 6: CONTINUE BUILD / DEPLOY
       ========================= */
    stage('Continue Build / Deploy') {
      steps {
        sh '''
          echo "Continuing build and deployment..."
          # deploy steps here
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/snyk-results/*.pdf', allowEmptyArchive: true
    }
  }
}
