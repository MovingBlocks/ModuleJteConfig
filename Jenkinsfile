// Minor housekeeping logic
boolean shouldPublish = env.BRANCH_NAME.equals("master") || env.BRANCH_NAME.equals("develop")

String[] jobNameParts = env.JOB_NAME.tokenize('/') as String[]
String realProjectName = jobNameParts.length < 2 ? env.JOB_NAME : jobNameParts[jobNameParts.length - 2]
boolean originNanoware = jobNameParts[0].equals("Nanoware")
// referring to the experimental line of Nanoware module build jobs on jenkins (jenkins.terasology.io/teraorg/job/Nanoware/job/TerasologyModules/job/X)
boolean experimental = jobNameParts.length >= 2 && jobNameParts[2].startsWith("X")

// Vary where we copy the build harness from based on where the actively running job lives
String buildHarnessOrigin = "Terasology/engine/develop"
if (originNanoware) {
    if (experimental) {
        // Unusual module tests with separate build harness (and JteConfig branch - defined elsewhere)
        buildHarnessOrigin = "Nanoware/Terasology/experimental"
    } else {
        // "Normal" module tests with the regular build harness in Nanoware land
        buildHarnessOrigin = "Nanoware/Terasology/develop"
    }
}

// Only keep a single build's worth of artifacts before truncating (module jars get published to Artifactory anyway)
properties([
    buildDiscarder(logRotator(artifactNumToKeepStr: '1'))
])

/**
 * Main pipeline definition for building Terasology modules.
 *
 * It uses the Declarative Pipeline Syntax.
 * See https://www.jenkins.io/doc/book/pipeline/syntax
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
pipeline {
    agent {
        label 'ts-module && heavy && java8'
    }
    stages {
         // declarative pipeline does `checkout scm` automatically when hitting first stage
        stage('Setup') {
            steps {
                echo 'Automatically checked out the things!'
                // the default resolution when omitting `defaultBranch` is to `master`
                // this is wrong in our case, so explicitly set `develop` as default
                // TODO: does this also work for PRs with different base branch?
                discoverGitReferenceBuild(defaultBranch: 'develop')

                echo "Copying in the build harness from an engine job: $buildHarnessOrigin"
                copyArtifacts(projectName: buildHarnessOrigin, filter: "templates/build.gradle", flatten: true, selector: lastSuccessful())
                copyArtifacts(projectName: buildHarnessOrigin, filter: "*, gradle/wrapper/**, config/**, natives/**, build-logic/**", selector: lastSuccessful())

                echo "Setting project name to: $realProjectName"

                sh 'rm -f gradle.properties'

                sh """
                    echo "rootProject.name = '$realProjectName'" > settings.gradle
                    echo 'includeBuild("build-logic")' >> settings.gradle
                """
                sh 'chmod +x gradlew'
            }
        }

        stage('Build') {
            steps {
                // Jenkins sometimes doesn't run Gradle automatically in plain console mode, so make it explicit
                sh './gradlew --console=plain clean htmlDependencyReport jar'
                archiveArtifacts 'build/libs/*.jar'
            }
            post {
                always {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/project/dependencies',
                        reportFiles: 'index.html',
                        reportName: 'Dependency Report',
                        reportTitles: 'Dependency Report'
                    ])
                }
            }
        }

        stage('Unit Tests') {
            steps {
                sh './gradlew --console=plain unitTest'
            }
            post {
                always {
                    // Gradle generates both a HTML report of the unit tests to `build/reports/tests/*`
                    // and XML reports to `build/test-results/*`.
                    // We need to upload the XML reports for visualization in Jenkins.
                    //
                    // See https://docs.gradle.org/current/userguide/java_testing.html#test_reporting
                    junit testResults: '**/build/test-results/unitTest/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('Publish') {
            when {
                expression {
                    shouldPublish
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'artifactory-gooey', \
                                                    usernameVariable: 'artifactoryUser', \
                                                    passwordVariable: 'artifactoryPass')]) {
                    sh '''./gradlew \\
                        --console=plain \\
                        -Dorg.gradle.internal.publish.checksums.insecure=true \\
                        publish \\
                        -PmavenUser=${artifactoryUser} \\
                        -PmavenPass=${artifactoryPass}
                    '''
                }
            }
        }

        stage('Analytics') {
            steps {
                sh './gradlew --console=plain check -x test'
            }
            post {
                always {
                    recordIssues skipBlames: true,
                    tools: [
                        checkStyle(pattern: '**/build/reports/checkstyle/*.xml'),
                        spotBugs(pattern: '**/build/reports/spotbugs/main/*.xml', useRankAsPriority: true),
                        pmdParser(pattern: '**/build/reports/pmd/*.xml')
                    ]

                    recordIssues skipBlames: true,
                        tool: taskScanner(includePattern: '**/*.java,**/*.groovy,**/*.gradle', \
                                        lowTags: 'WIBNIF', normalTags: 'TODO', highTags: 'FIXME')
                }
            }
        }

        stage('Documentation') {
            steps {
                sh './gradlew --console=plain javadoc'
                script {
                    if (fileExists("build/docs/javadoc/index.html")) {
                        javaDoc javadocDir: 'build/docs/javadoc', keepAll: false
                        recordIssues skipBlames: true, tool: javaDoc()
                    }
                }
            }
            post {
                always {
                    publishHTML([
                        allowMissing: true,
                        keepAll: false,
                        alwaysLinkToLastBuild: true,
                        reportDir: 'docs',
                        reportFiles: 'index.html',
                        reportName: 'Docs',
                        reportTitles: 'Docs'
                    ])
                }
            }
        }

        stage('Integration Tests') {
            steps {
                sh './gradlew --console=plain integrationTest'
            }
            post {
                always {
                    // Gradle generates both a HTML report of the unit tests to `build/reports/tests/*`
                    // and XML reports to `build/test-results/*`.
                    // We need to upload the XML reports for visualization in Jenkins.
                    //
                    // See https://docs.gradle.org/current/userguide/java_testing.html#test_reporting
                    junit testResults: '**/build/test-results/integrationTest/*.xml', allowEmptyResults: true, healthScaleFactor: 0.0
                }
            }
        }
    }
}
