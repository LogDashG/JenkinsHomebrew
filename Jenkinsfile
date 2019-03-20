#!/usr/bin/env groovy
/*
 * Jenkinsfile
 * JenkinsHomebrew
 *
 * Checks for new versions of Jenkins, updating the core formula when found.
 */

import jenkins.model.*
import hudson.model.Result

def MAIL_TO = JenkinsLocationConfiguration.get().getAdminAddress()
echo "MAIL_TO: $MAIL_TO"

properties([
    // Don't trigger this job when changes are found from branch indexing.
    //overrideIndexTriggers(false),
    disableConcurrentBuilds(),
    pipelineTriggers([
        cron('H 1 * * *')
    ]),
    buildDiscarder(logRotator(numToKeepStr: '100')),
])

// Global variables
String currentVersion
String newVersion
String file
String url
String hash

try {
    timeout(time: 1, unit: 'HOURS') {
        withEnv(['LANG=en_US.UTF-8']) {
            node {
                stage("💁🏻‍♀️ Current Version") {
                    // Parse version from homebrew formula json
                    // https://jenkins.io/doc/pipeline/steps/workflow-durable-task-step/#sh-shell-script
                    currentVersion = sh(
                        script: """
                            brew info --json=v1 jenkins \
                                | jq .[0].versions.stable \
                                | tr -d '"'
                        """,
                        label: "✨ Version",
                        returnStdout: true
                    ).trim()

                    echo "Current Jenkins formula version: $currentVersion"
                }
                stage("📡 New Release") {
                    // Increment minor version
                    def (major, minorString) = currentVersion.tokenize('.')
                    Integer minor = minorString as Integer
                    minor++
                    newVersion = "$major.$minor"

                    echo "Checking for new Jenkins release with version: $newVersion"
                    file = "jenkins-${newVersion}.war"
                    url = "http://mirrors.jenkins.io/war/$newVersion/jenkins.war"

                    Integer status = sh(
                        script: """
                            curl \
                                --head \
                                --fail \
                                --silent \
                                $url
                        """,
                        label: "✔️ Check",
                        returnStatus: true
                    )
                    if (status != 0) {
                        echo "Version $currentVersion is still the latest. 😞"
                        currentBuild.rawBuild.@result = hudson.model.Result.ABORTED
                    } else {
                        echo "Jenkins $newVersion IS NOW AVAILABLE! 🎉"
                    }
                }

                echo "currentBuild.result: ${currentBuild.result}"

                // build.@result = hudson.model.Result.SUCCESS
                // build.@result = hudson.model.Result.NOT_BUILT
                // build.@result = hudson.model.Result.UNSTABLE
                // build.@result = hudson.model.Result.FAILURE
                // build.@result = hudson.model.Result.ABORTED

                if (currentBuild.result && currentBuild.result != 'SUCCESS') {
                    return
                }

                stage("👇🏻 Download") {
                    echo "👇🏻 Downloading Jenkins $newVersion - $url"

                    sh(
                        script: """
                            curl \
                                --fail \
                                --location \
                                --silent \
                                --output $file \
                                $url
                        """,
                        label: "👇🏻 Download"
                    )
                    hash = sh(
                        script: """
                            shasum \
                                --algorithm 256 \
                                $file \
                            | cut -d' ' -f1
                        """,
                        label: "🔢 Hash",
                        returnStdout: true
                    ).trim()

                    echo "File hash: $hash"
                }
                stage("✉️ Notify") {
                    mail to: MAIL_TO,
                        subject: "✨👷🏻‍♂️ Jenkins $newVersion has been released",
                        body: """
                            Jenkins $newVersion
                            $url
                            $hash
                        """
                }
            } // node

            // Intentionally run outside of a node block because: It is a waste of an executor slot to wrap the build
            // step in node. Your upstream executor will just be sitting idle for no reason.
            stage("🍼 Formula") {
                // https://jenkins.io/doc/pipeline/steps/pipeline-build-step/#build-build-a-job
                build job: "../JenkinsPhatblatServices/master",
                    parameters: [
                        string(name: "newVersion", value: newVersion),
                        string(name: "fileHash", value: hash)
                    ]
            }
        }
    }
}
catch (Exception ex) {
    String jobName = env.JOB_NAME
    String buildNumber = env.BUILD_NUMBER

    String subject = "👷🏻‍♂️ [FAILURE] $jobName - Build #$buildNumber"
    String body = """$jobName
                |Build #${buildNumber} - FAILURE
                |
                |${env.BUILD_URL}
                |
                |${ex}
                |
                |${env.BUILD_URL}console
                |""".stripMargin()

    mail to: MAIL_TO,
         subject: subject,
         body: body
    throw ex
}
