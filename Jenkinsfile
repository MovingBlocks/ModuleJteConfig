// Only keep a single build's worth of artifacts before truncating (module jars get published to Artifactory anyway)
properties([
    buildDiscarder(logRotator(artifactNumToKeepStr: '1'))
])

/**
 * Main pipeline definition for building Terasology modules.
 *
 * It uses the Scripted Pipeline Syntax.
 * See https://www.jenkins.io/doc/book/pipeline/#declarative-versus-scripted-pipeline-syntax
 *
 * This pipeline uses Jenkins plugins to collect and report additional information about the build.
 *
 *   - Warnings Next Generation Plugin (https://plugins.jenkins.io/warnings-ng/)
 *      To record issues from code scans and static analysis tools, e.g., CheckStyle or Spotbugs.
 *   - Git Forensics Plugin (https://plugins.jenkins.io/git-forensics/)
 *      To compare code scans and static analysis against a reference build from the base branch.
 *   - JUnit Plugin (https://plugins.jenkins.io/junit/)
 *      To record the results of our test suites from JUnit format. 
 *
 */
node ("ts-module && heavy && java8") {
    stage('Setup') {
        echo "Going to check out the things !"
        checkout scm

        // Vary where we copy the build harness from based on where the actively running job lives
        def buildHarnessOrigin = "Terasology/engine/develop"
        if (env.JOB_NAME.startsWith("Nanoware/TerasologyModules/H")) { // "Normal" module tests with the regular build harness in Nanoware land
            buildHarnessOrigin = "Nanoware/Terasology/develop"
        } else if (env.JOB_NAME.startsWith("Nanoware/TerasologyModules/X")) { // Unusual module tests with separate build harness (and JteConfig branch - defined elsewhere)
            buildHarnessOrigin = "Nanoware/Terasology/experimental"
        }

        echo "Copying in the build harness from an engine job: $buildHarnessOrigin"
        copyArtifacts(projectName: buildHarnessOrigin, filter: "templates/build.gradle", flatten: true, selector: lastSuccessful())
        copyArtifacts(projectName: buildHarnessOrigin, filter: "*, gradle/wrapper/**, config/**, natives/**, build-logic/**", selector: lastSuccessful())

        def realProjectName = findRealProjectName()
        echo "Setting project name to: $realProjectName"

        sh 'rm -f gradle.properties'

        sh """
            echo "rootProject.name = '$realProjectName'" > settings.gradle
            echo 'includeBuild("build-logic")' >> settings.gradle
        """

        sh 'chmod +x gradlew'
    }

    stage('Build') {
        sh './gradlew --console=plain clean htmlDependencyReport jar'
        publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'build/reports/project/dependencies', reportFiles: 'index.html', reportName: 'Dependency Report', reportTitles: 'Dependency Report'])
        archiveArtifacts 'build/libs/*.jar'
    }

    stage('Unit Tests') {
        try {
            sh './gradlew --console=plain unitTest'
        } finally {
            junit testResults: '**/build/reports/tests/unitTest/*.xml', allowEmptyResults: true
        }
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
        sh './gradlew --console=plain check -x test'
        // the default resolution when omitting `defaultBranch` is to `master` - which is wrong in our case. 
        discoverGitReferenceBuild(defaultBranch: 'develop') //TODO: does this also work for PRs with different base branch?

        recordIssues skipBlames: true,
            tools: [
                checkStyle(pattern: '**/build/reports/checkstyle/*.xml'),
                spotBugs(pattern: '**/build/reports/spotbugs/main/*.xml', useRankAsPriority: true),
                pmdParser(pattern: '**/build/reports/pmd/*.xml')
            ] 

        recordIssues skipBlames: true, 
            tool: taskScanner(includePattern: '**/*.java,**/*.groovy,**/*.gradle', lowTags: 'WIBNIF', normalTags: 'TODO', highTags: 'FIXME')
    }

    stage('Documentation') {
        sh './gradlew --console=plain javadoc'
        // Test for the presence of Javadoc so we can skip it if there is none (otherwise would fail the build)
        if (fileExists("build/docs/javadoc/index.html")) {
            step([$class: 'JavadocArchiver', javadocDir: 'build/docs/javadoc', keepAll: false])
            recordIssues skipBlames: true, tool: javaDoc()
        }
    }

    stage('Integration Tests') {
        try {
            sh './gradlew --console=plain integrationTest'
        } catch (err) {
            // Currently, all MTE tests are broken after the migration to gestalt v7.
            // 
            // By catching the error here the failed test execution should not fail the stage,
            // and thereby no longer fail the complete build.
            //
            // See https://github.com/MovingBlocks/Terasology/issues/4757
        } finally {
            junit testResults: '**/build/reports/tests/integrationTest/*.xml', allowEmptyResults: true, healthScaleFactor: 0.0
        }
    }
}

String findRealProjectName() {
    def jobNameParts = env.JOB_NAME.tokenize('/') as String[]
    println "Job name parts: $jobNameParts"
    return jobNameParts.length < 2 ? env.JOB_NAME : jobNameParts[jobNameParts.length - 2]
}
