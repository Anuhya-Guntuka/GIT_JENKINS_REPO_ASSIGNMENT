pipeline {
    agent any

    stages {

        stage('Build Info') {
            steps {
                sh 'hostname'
                sh 'whoami'
            }
        }

        stage('Maven Build') {
            steps {
                sh 'mvn clean package'
            }
        }

    }
}
