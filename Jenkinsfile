// Only keep a single build's worth of artifacts before truncating (module jars get published to Artifactory anyway)
properties([
    buildDiscarder(logRotator(artifactNumToKeepStr: '1'))
])

node ("ts-module && heavy && java8") {
    stage('Prepare') {
        echo "Going to check out the things !"
        checkout scm

        echo "Copying in the build harness from an engine job"
        copyArtifacts(projectName: "Terasology/engine/develop", filter: "templates/build.gradle", flatten: true, selector: lastSuccessful())
        copyArtifacts(projectName: "Terasology/engine/develop", filter: "*, gradle/wrapper/**, config/**, natives/**, build-logic/**", selector: lastSuccessful())

        def realProjectName = findRealProjectName()
        echo "Setting real project name to: $realProjectName"
        sh """
            ls
            rm -f settings.gradle
            rm -f gradle.properties
            echo "rootProject.name = '$realProjectName'" >> settings.gradle
            echo 'includeBuild("build-logic")' >> settings.gradle
            cat settings.gradle
            chmod +x gradlew
        """
    }

    stage('Build') {
        sh './gradlew clean htmlDependencyReport jar'
        publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'build/reports/project/dependencies', reportFiles: 'index.html', reportName: 'Dependency Report', reportTitles: 'Dependency Report'])
        archiveArtifacts 'build/libs/*.jar'
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

    stage('Analytics') {
        sh './gradlew check spotbugsmain javadoc'
    }

    stage('Record') {
        // Test for the presence of Javadoc so we can skip it if there is none (otherwise would fail the build)
        if (fileExists("build/docs/javadoc/index.html")) {
            step([$class: 'JavadocArchiver', javadocDir: 'build/docs/javadoc', keepAll: false])
            recordIssues tool: javaDoc()
        }
        junit testResults: 'build/test-results/test/*.xml', allowEmptyResults: true, healthScaleFactor: 0.0
        recordIssues tool: checkStyle(pattern: '**/build/reports/checkstyle/*.xml')
        recordIssues tool: spotBugs(pattern: '**/build/reports/spotbugs/main/*.xml', useRankAsPriority: true)
        recordIssues tool: pmdParser(pattern: '**/build/reports/pmd/*.xml')
        recordIssues tool: taskScanner(includePattern: '**/*.java,**/*.groovy,**/*.gradle', lowTags: 'WIBNIF', normalTags: 'TODO', highTags: 'ASAP')
    }
}

String findRealProjectName() {
    def jobNameParts = env.JOB_NAME.tokenize('/') as String[]
    println "Job name parts: $jobNameParts"
    return jobNameParts.length < 2 ? env.JOB_NAME : jobNameParts[jobNameParts.length - 2]
}
