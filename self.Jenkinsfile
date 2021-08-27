// This repo exports `Jenkinsfile` for module builds.
// This file is to do some very basic checks on that file, to catch errors earlier.

pipeline {
    agent { label 'master' }
    stages {
        stage('Syntax Check') {
            steps {
                script {
                    boolean isValid = validateDeclarativePipeline 'Jenkinsfile'
                    if (!isValid) {
                        error("Validation failed.")
                    }
                }
            }
        }
    }
}
