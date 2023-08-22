#!/usr/bin/env groovy

@Library('jenkins-bops')
import bops.util.AppImage

gradleTools   = new bops.util.Gradle()
slackTools    = new bops.util.Slack()
jenkinsTools  = new bops.util.Jenkins()
branchToBuild = env.BRANCH_NAME

if (params.branch) {
    if (params.branch.startsWith('origin/')) {
        branchToBuild = "${(params.branch).split('/')[1..-1].join('/')}"
    } else {
        branchToBuild = "${params.branch}"
    }
}

appImage = new AppImage(script:     this,
                        appName:    'rma',
                        repo:       'JS-RMA',
                        repoBranch: branchToBuild,
                        repoKey:    'github-AIGEXP',
                        account:    'AIGEXP',
                        vertical:   'jservices',
                        dockerfile: 'infrastructure/rma/Dockerfile')

String getVersion(){
    version = sh(
        script: """
            cat ${appImage.getRepo()}/app/src/main/resources/application.yml \
            | grep version \
            | cut -d ' ' -f2
        """,
        returnStdout: true
    ).trim()

    if(version)
        return version

    date = new Date()
    tz   = TimeZone.getTimeZone('UTC')

    return date.format('yyyyMMdd', tz) + '-' + date.format('HHmmss', tz)
}

pipeline {
    agent {
        kubernetes {
            defaultContainer 'builder'
            inheritFrom 'builder-java17'
        }
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        skipDefaultCheckout()
    }

    environment {
        REPO_NAME         = appImage.getRepo()
        NEXUS_URL         = 'http://nexus-dev-services.jumia.com:8080'
        NEXUS_CREDENTIALS = credentials('nexus')
    }

    stages {
        stage('Checkout Repository') {
            steps {
                script {
                    appImage.checkoutToSubdir(REPO_NAME)
                    version                  = getVersion()
                    appImage.version         = version
                    currentBuild.description = "${version} from ${appImage.getRepoBranch()}"
                }
            }
        }

        stage('Build Dockerfile') {
            steps {
                sleep(10) // due to: unable to resolve docker endpoint: open /certs/ca.pem: no such file or directory

                dir(REPO_NAME) {
                    script {
                        sh '''
                            sed -i "s|REPO_URL|${NEXUS_URL}|" gradle.properties
                            sed -i "s|REPO_USERNAME|${NEXUS_CREDENTIALS_USR}|" gradle.properties
                            sed -i "s|REPO_PASSWORD|${NEXUS_CREDENTIALS_PSW}|" gradle.properties
                        '''
                    }
                }

                script {
                    appImage.buildDockerImage()
                }
            }
        }

        stage('Push Docker Image') {
            when {
                expression { branch 'master' }
            }
            steps {
                script {
                    appImage.pushDockerImage()
                }
            }
        }
    }

    post {
        success {
            script {
                if (env.BRANCH_NAME == 'master'){
                    build job: "OMS/deploy/staging/rma",
                          parameters: [
                            string(name: "version", value: "jumiaservices/rma:${appImage.getVersion()}"),
                            string(name: "namespaces", value: "glb"),
                            string(name: "name", value: "rma"),
                            string(name: "rolloutElement", value: "deployment/rma")],
                          wait: false,
                          propagate: false
                }
            }
        }
    }
}

