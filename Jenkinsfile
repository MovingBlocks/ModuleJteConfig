node ("default-java") {

    stage('Prepare') {
        echo "Going to check out the things !"
        checkout scm

        echo "Copying in the build harness from an engine job"
        copyArtifacts(projectName: "Terasology/engine/develop", filter: "templates/build.gradle", flatten: true, selector: lastSuccessful())
        copyArtifacts(projectName: "Terasology/engine/develop", filter: "*, gradle/wrapper/**, config/**, natives/**", selector: lastSuccessful())

        def realProjectName = findRealProjectName()
        echo "Setting real project name to: $realProjectName"
        sh """
            ls
            rm -f settings.gradle
            rm -f gradle.properties
            echo "rootProject.name = '$realProjectName'" >> settings.gradle
            cat settings.gradle
            chmod +x gradlew
        """
    }
    
    stage('Build') {
        sh './gradlew clean jar'
        archiveArtifacts 'gradlew, gradle/wrapper/*, templates/build.gradle, config/**, build/distributions/Terasology.zip, build/resources/main/org/terasology/version/versionInfo.properties, natives/**'
    }
    
    stage('Test') {
        // Keep tests in a separate stage to cope with failing MTE
        sh './gradlew test'
    }

    stage('Analytics') {
        // Run analytics like Checkstyle or PMD without running tests
        sh './gradlew check -x test'
    }
    
    stage('JavaDoc') {
        sh './gradlew javadoc'
    }
    
    stage('Publish') {
        if (env.BRANCH_NAME.equals("master") || env.BRANCH_NAME.equals("develop")) {
            withCredentials([usernamePassword(credentialsId: 'artifactory-gooey', usernameVariable: 'artifactoryUser', passwordVariable: 'artifactoryPass')]) {
                sh './gradlew --console=plain -Dorg.gradle.internal.publish.checksums.insecure=true publish -PmavenUser=${artifactoryUser} -PmavenPass=${artifactoryPass}'
            }
        } else {
            println "Running on a branch other than 'master' or 'develop' bypassing publishing"
        }
    }
    
    stage('Record') {
        // Test for the presence of Javadoc so we can skip it if there is none (otherwise would fail the build)
        if (fileExists("build/docs/javadoc/index.html")) {
            step([$class: 'JavadocArchiver', javadocDir: 'build/docs/javadoc', keepAll: false])
            recordIssues tool: javaDoc()
        }
        //TODO: Figure out how to collect test results without failing the build/making it unstable
        //      Don't collect the tests results as this will (most often) make the build UNSTABLE, which in turn
        //      will turn the Github status to be failed.
        // junit testResults: 'build/test-results/test/*.xml', allowEmptyResults: true, healthScaleFactor: 0.0
        recordIssues tool: checkStyle(pattern: '**/build/reports/checkstyle/*.xml')
        recordIssues tool: spotBugs(pattern: '**/build/reports/spotbugs/*.xml', useRankAsPriority: true)
        recordIssues tool: pmdParser(pattern: '**/build/reports/pmd/*.xml')
        recordIssues tool: taskScanner(includePattern: '**/*.java,**/*.groovy,**/*.gradle', lowTags: 'WIBNIF', normalTags: 'TODO', highTags: 'ASAP')

        //TODO: This makes a second check on Github instead of updating the job-related one
        //      Mark UNSTABLE builds as SUCCESS on Github.
        step([ 
            $class: 'GitHubCommitStatusSetter', 
            statusResultSource: [
                $class: 'ConditionalStatusResultSource', 
                results: [[$class: 'BetterThanOrEqualBuildResult', message: '', result: 'UNSTABLE', state: 'SUCCESS']]
            ]
        ])

    }
}

String findRealProjectName() {
    def jobNameParts = env.JOB_NAME.tokenize('/') as String[]
    println "Job name parts: $jobNameParts"
    return jobNameParts.length < 2 ? env.JOB_NAME : jobNameParts[jobNameParts.length - 2]
}
