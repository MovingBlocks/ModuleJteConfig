pipeline {
    agent {
        label 'master'
    }
    stages {
        stage('Syntax Check') {
            steps {
                validateDeclarativePipeline 'Jenkinsfile'
            }
        }
    }
}
