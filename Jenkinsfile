pipeline {
    agent { label 'mule-builder' }

    options {
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target deployment environment')
        string(name: 'APP_NAME', defaultValue: 'my-mule-app', description: 'CloudHub application name')

    }

    environment {
        ANYPOINT_CREDS  = credentials('anypoint-connected-app')
        MAVEN_OPTS      = '-Xmx1024m'

        MAVEN_LOCAL_REPO = '/home/jenkins/.m2/repository'
    }

    stages {

        stage('Build & Unit Test') {
            steps {
                sh 'mvn clean package -DskipMunitTests -Dmaven.repo.local=${MAVEN_LOCAL_REPO}'
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('MUnit Tests') {
            steps {
                sh 'mvn test -Dmaven.repo.local=${MAVEN_LOCAL_REPO}'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                    publishHTML(target: [
                        reportDir: 'target/site/munit/coverage',
                        reportFiles: 'summary.html',
                        reportName: 'MUnit Coverage Report',
                        keepAll: true
                    ])
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                expression { params.ENVIRONMENT in ['dev', 'staging', 'prod'] }
            }
            steps {
                deployToCloudHub('dev')
            }
        }

        stage('Deploy to Staging') {
            when {
                expression { params.ENVIRONMENT in ['staging', 'prod'] }
            }
            steps {
                deployToCloudHub('staging')
            }
        }

        stage('Approval for Production') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                timeout(time: 24, unit: 'HOURS') {
                    input message: "Deploy ${params.APP_NAME} to PRODUCTION?", ok: 'Approve', submitter: 'release-approvers'
                }
            }
        }

        stage('Deploy to Production') {
            when {
                expression { params.ENVIRONMENT == 'prod' }
            }
            steps {
                deployToCloudHub('prod')
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded: ${params.APP_NAME} deployed to ${params.ENVIRONMENT}"
        }
        failure {
            echo "Pipeline FAILED for ${params.APP_NAME} targeting ${params.ENVIRONMENT}"
            // Replace with your notification of choice:
            // slackSend channel: '#deployments', color: 'danger', message: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            // emailext subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}", to: 'team@example.com', body: "Check: ${env.BUILD_URL}"
        }
        always {
            cleanWs()
        }
    }
}

def deployToCloudHub(String targetEnv) {
    def envSuffix = (targetEnv == 'prod') ? '' : "-${targetEnv}"
    def appName   = "${params.APP_NAME}${envSuffix}"

    retry(2) {
        sh """
            mvn deploy -DmuleDeploy \
                -Dmaven.repo.local=\${MAVEN_LOCAL_REPO} \
                -Dclient.id=\${ANYPOINT_CREDS_USR} \
                -Dclient.secret=\${ANYPOINT_CREDS_PSW} \
                -Dapp.name=${appName} \
                -Denv=${targetEnv} \
                -DskipTests
        """
    }
    echo "Deployed ${appName} to ${targetEnv}"
}
