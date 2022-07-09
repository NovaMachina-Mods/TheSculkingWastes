pipeline {
    agent {
      label 'master'
    }
    environment {
        NEXUS_USERNAME = credentials('MavenUser')
        NEXUS_PASSWORD = credentials('MavenPassword')
        CURSEFORGE_KEY = credentials('CurseForgeAPIKey')
        DISCORD_WEBHOOK_URL = credentials('discord-webhook-url')
        DISCORD_PREFIX = "Job: The Sculking Wastes Branch: ${BRANCH_NAME} Build: #${BUILD_NUMBER}"
    }
    options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    }
    stages {
        stage('Notify Start') {
            when {
                not {
                    changeRequest()
                }
            }
            steps {
                discordSend(
                    title: "${DISCORD_PREFIX} Started",
                    successful: true,
                    result: 'ABORTED',
                    webhookURL: DISCORD_WEBHOOK_URL
                )
            }
        }
        stage('Build') {
            steps {
                sh 'chmod +x gradlew'
                withGradle {
                    sh './gradlew clean build'
                }
            }
        }
        stage('SonarQube Analysis') {
          steps {
            withSonarQubeEnv(installationName: "SonarQube Integration", credentialsId: 'SonarQube Admin Token') {
              sh "./gradlew sonarqube"
            }
          }
        }
        stage("Quality Gate") {
          steps {
            timeout(time: 1, unit: 'HOURS') {
              // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
              // true = set pipeline to UNSTABLE, false = don't
              waitForQualityGate(webhookSecretId: 'SonarQubeWebhookSecret', abortPipeline: true)
            }
          }
        }
        stage('Get Artifacts to Build') {
            steps {
                script {
                    env.DEPLOY_API = sh (
                        script: 'git diff --stat --name-only ${GIT_PREVIOUS_SUCCESSFUL_COMMIT} ${GIT_COMMIT} | grep -q src/api',
                        returnStatus: true
                    ) == 0
                    env.DEPLOY_MAIN = sh (
                        script: 'git diff --stat --name-only ${GIT_PREVIOUS_SUCCESSFUL_COMMIT} ${GIT_COMMIT} | grep -q src/main || \
                                 git diff --stat --name-only ${GIT_PREVIOUS_SUCCESSFUL_COMMIT} ${GIT_COMMIT} | grep -q src/datagen/main || \
                                 git diff --stat --name-only ${GIT_PREVIOUS_SUCCESSFUL_COMMIT} ${GIT_COMMIT} | grep -q src/datagen/generated/sculkingwastes',
                        returnStatus: true
                    ) == 0
                    echo "Should deploy API: ${DEPLOY_API}"
                    echo "Should deploy MAIN: ${DEPLOY_MAIN}"
                }
            }
        }
        stage('Deploy Packages') {
            when {
                branch '1.19.X'
            }
            parallel {
                stage('API') {
                    when {
                        expression {
                            env.DEPLOY_API == 'true'
                        }
                    }
                    steps {
                        withGradle {
                            sh './gradlew publishApiPublicationToMavenRepository'
                        }
                    }
                }
                stage('Main') {
                    when {
                        expression {
                            env.DEPLOY_MAIN == 'true'
                        }
                    }
                    steps {
                        withGradle {
                            sh './gradlew curseforge400012 publishMainPublicationToMavenRepository'
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                if(env.CHANGE_ID == null) {
                    discordSend(
                        title: "${DISCORD_PREFIX} Finished ${currentBuild.currentResult}",
                        successful: currentBuild.resultIsBetterOrEqualTo("SUCCESS"),
                        result: currentBuild.currentResult,
                        webhookURL: DISCORD_WEBHOOK_URL
                    )
                }
            }
        }
    }
}