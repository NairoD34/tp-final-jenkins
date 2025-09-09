@Library('jenkins-shared-lib') _

pipeline {
    agent any
    parameters {
        choice(name: 'deploy_env', choices: ['staging', 'production'], description: 'Environnement de déploiement')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build & Test') {
            parallel {
                stage('Backend') {
                    agent { label 'build-heavy' }
                    steps {
                        sh 'mvn clean package'
                        junit 'target/surefire-reports/*.xml'
                    }
                }
                stage('Frontend') {
                    steps {
                        sh 'npm install'
                        sh 'npm test -- --ci --reporters=jest-junit'
                        junit 'front/junit.xml'
                    }
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh 'mvn sonar:sonar'
                    }
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Security') {
            parallel {
                stage('Dependency Check') {
                    steps {
                        sh 'mvn dependency-check:check || npm audit --audit-level=high'
                    }
                }
                stage('Docker Scan') {
                    steps {
                        sh 'trivy image bookmymovie:latest --exit-code 1 --severity CRITICAL'
                    }
                }
            }
        }
        stage('Package & Artefacts') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar, front/build/**', fingerprint: true
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        def img = docker.build("bookmymovie:${env.BUILD_NUMBER}")
                        img.push()
                    }
                }
            }
        }
        stage('Deploy to Staging') {
            when { expression { params.deploy_env == 'staging' } }
            steps {
                sh 'docker-compose -f docker-compose.staging.yml up -d'
                notifySlack("Déploiement staging réussi")
            }
        }
        stage('Manual Approval') {
            when { expression { params.deploy_env == 'production' } }
            steps {
                input message: 'Déployer en production ?'
            }
        }
        stage('Deploy to Production') {
            when { expression { params.deploy_env == 'production' } }
            steps {
                sh 'kubectl apply -f k8s/prod/'
                notifySlack("Déploiement production réussi")
            }
        }
    }
    post {
        always {
            notifyEmail()
        }
    }
}