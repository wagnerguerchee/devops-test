pipeline {
    agent any
    
    environment {
        APP = "app-teste"
        SCANNER_HOME = tool 'sonar-scanner'
        DISCORD_WEBHOOK = 'https://discord.com/api/webhooks/1'
        SONARQUBE_URL = 'http://10.10.10.52:9000/dashboard?id=app
    }
    
    stages {
        stage('Exibir Variáveis da Branch') {
            steps {
                script {
                    BRANCH_NAME = env.GIT_BRANCH?.replaceAll('origin/', '') ?: 'unknown'
                    echo "BRANCH_NAME: ${BRANCH_NAME}"
                }
            }
        }

        stage('Configurar Versão') {
            steps {
                script {
                    switch (BRANCH_NAME) {
                        case 'master':
                            VERSION = 'prod'
                            break
                        case 'homolog':
                            VERSION = 'hml'
                            break
                        default:
                            VERSION = 'other'
                            echo "Branch não suportada para construção: ${BRANCH_NAME}"
                            return
                    }
                }
            }
        }

        stage('Construir e Empurrar Docker Image') {
            steps {
                script {
                    def DOCKERFILE_PATH = BRANCH_NAME == 'homolog' ? "./Dockerfiles/Homologation/Dockerfile" : "./Dockerfiles/Production/Dockerfile"
                    def imageName = "devops/${APP}-${VERSION}-${env.BUILD_NUMBER}"
                    dockerapp = docker.build(imageName, "-f ${DOCKERFILE_PATH} ./") 
                }
            }
        }

        stage('SonarQube Analysis') {
            when {
                expression { BRANCH_NAME == 'homolog' }
            }
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                        sh "sudo /home/desenvolvimento/.dotnet/tools/dotnet-sonarscanner begin /k:'evaluation' /d:sonar.host.url='http://10.10.10.52:9000' /d:sonar.login='squ_3982b7bd5296ccfd' /d:sonar.cs.opencover.reportsPaths='app/coverage.opencover.xml' /d:sonar.coverage.exclusions='app.WebApi/**'"
                        sh "sudo dotnet build"
                        sh "sudo dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover /p:CoverletOutput=coverage.opencover.xml"
                        sh "sudo dotnet-sonarscanner end /d:sonar.login='squ_3982b7bd5296cc'"
                    }
                }

        stage('Quality gate') {
            when {
                expression { BRANCH_NAME == 'homolog' }
            }
            steps {
                script {
                    timeout(time: 1, unit: 'HOURS') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        }
                    }
                }
            }
            post {
                failure {
                    echo 'Quality gate: Failed Cobertura de Código abaixo de 90 %'
                    discordSend(
                        description: """Pipeline falhou na branch ${env.GIT_BRANCH}.
Veja mais detalhes em ${env.BUILD_URL}.
Acesse o SonarQube para mais informações: ${SONARQUBE_URL}.""",
                        footer: "Pipeline falhou no Quality Gate",
                        link: env.BUILD_URL,
                        result: currentBuild.currentResult, 
                        title: APP,
                        webhookURL: DISCORD_WEBHOOK
                    )
                }
            }
        }

        stage('Upload Docker Image') {
            steps {
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: 'nexus-user', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]){
                            sh "docker login -u $USERNAME -p $PASSWORD ${NEXUS_URL}"
                            sh "docker tag devops/${APP}-${VERSION}-${env.BUILD_NUMBER} ${NEXUS_URL}/devops/${APP}-${VERSION}:${env.BUILD_NUMBER}"
                            sh "docker push ${NEXUS_URL}/devops/${APP}-${VERSION}:${env.BUILD_NUMBER}"
                        }
                    } catch (Exception e) {
                        error "Erro durante o upload da imagem: ${e.message}"
                    }
                }
            }
        }

        stage('Trigger Job') {
            steps {
                script {
                    build job: 'update-manifest-evaluation', parameters: [
                        text(name: 'DOCKERTAG', value: env.BUILD_NUMBER),
                        text(name: 'VERSION', value: BRANCH_NAME)
                    ]      
                }
            }
        }

    post {
        always {
            script {
                def resultMessage = currentBuild.currentResult == 'SUCCESS' ? 'Sucesso' : 'Falha'
                discordSend(
                    description: """Pipeline concluído com resultado: ${resultMessage} na branch ${BRANCH_NAME}.
Veja mais detalhes em ${env.BUILD_URL}.
Acesse o SonarQube para mais informações: ${SONARQUBE_URL}.""",
                    footer: resultMessage,
                    link: env.BUILD_URL,
                    result: currentBuild.currentResult, 
                    title: APP,
                    webhookURL: DISCORD_WEBHOOK
                )
            }
        }
    }