#!/usr/bin/env groovy
/*
 * Jenkinsfile
 * JenkinsHomebrew
 *
 * Checks for new versions of Jenkins, updating the core formula when found.
 */

import jenkins.model.*

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


String version = "2.167"
String file = "jenkins-$version.war"
String url = "http://mirrors.jenkins.io/war/$version/jenkins.war"

try {
    timeout(time: 1, unit: 'HOURS') {
        withEnv(['LANG=en_US.UTF-8']) {
            node {
                stage("✨ Latest Version") {
                    // https://jenkins.io/doc/pipeline/steps/workflow-durable-task-step/#sh-shell-script
                    String output = sh(
                        script: "brew info --json=v1 jenkins | jq .[0].versions.stable",
                        label: "✨ Version",
                        returnStdout: true
                    ).trim()
                    echo "Jenkins formula version: $output"
                    // TODO: Parse version
                }
                stage("📡 Check for new") {
                    // TODO: Compare versions
                }
                stage("👇🏻 Download") {
                    echo "👇🏻 Downloading Jenkins $version - $url"
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
                    sh(
                        script: """
                            shasum \
                                --algorithm 256 \
                                $file
                        """,
                        label: "🔢 Hash",
                        returnStdout: true
                    ).trim()
                }
                stage("🍼 Formula") {

                }
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
                |${ex.message}
                |
                |${env.BUILD_URL}/console
                |""".stripMargin()

    mail to: MAIL_TO,
         subject: subject,
         body: body
    throw ex
}
