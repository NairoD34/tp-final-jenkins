pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Installation des dépendances'
                sh 'npm install'
            }
        }
        stage('Test') {
            steps {
                echo 'Exécution des tests unitaires'
                sh 'npm test'
            }
        }
    }
}