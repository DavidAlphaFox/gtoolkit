import hudson.tasks.test.AbstractTestResultAction
import hudson.model.Actionable
import hudson.tasks.junit.CaseResult

@NonCPS
def getFailedTests = { ->
    def testResultAction = currentBuild.rawBuild.getAction(AbstractTestResultAction.class)
    def failedTestsString = "```"

    if (testResultAction != null) {
        def failedTests = testResultAction.getFailedTests()

        if (failedTests.size() > 9) {
            failedTests = failedTests.subList(0, 8)
        }

        for(CaseResult cr : failedTests) {
            failedTestsString = failedTestsString + "${cr.getFullDisplayName()}:\n${cr.getErrorDetails()}\n\n"
        }
        failedTestsString = failedTestsString + "```"
    }
    return failedTestsString
}

@NonCPS
def getTestSummary = { ->
    def testResultAction = currentBuild.rawBuild.getAction(AbstractTestResultAction.class)
    def summary = ""

    if (testResultAction != null) {
        total = testResultAction.getTotalCount()
        failed = testResultAction.getFailCount()

        summary = "Passed: " + (total - failed)
        summary = summary + (", Failed: " + failed)
    } else {
        summary = "No tests found"
    }
    return summary
}

pipeline {
    agent none
    parameters { string(name: 'FORCED_TAG_NAME', defaultValue: '', description: 'Environment variable used for increasing the minor or major version of Glamorous Toolkit. Example input `v0.8.0`. Can be left blank, in that case, the patch will be incremented.') }
    options { 
        buildDiscarder(logRotator(numToKeepStr: '50'))
        disableConcurrentBuilds() 
    }
    environment {
        GITHUB_TOKEN = credentials('githubrelease')
        AWSIP = 'ec2-18-197-145-81.eu-central-1.compute.amazonaws.com'
        MASTER_WORKSPACE = ""
        EXAMPLE_PACKAGES = "GToolkit-.* GT4SmaCC-.* DeepTraverser-.* Brick Brick-.* Bloc Bloc-.* Sparta-.*"
    }
    stages {
        stage ('Build pre release') {
            agent {
                label "unix"
            }
            stages {
                stage('Load latest master commit') {
                    when { expression {
                            env.BRANCH_NAME.toString().equals('master')
                        }
                    }
                    steps {
                        script {
                            MASTER_WORKSPACE = WORKSPACE
                        }
                        sh 'git clean -fdx'
                        sh 'chmod +x scripts/build/*.sh'
                        sh 'rm -rf pharo-local/iceberg'
                        
                        slackSend (color: '#FFFF00', message: ("Started <${env.BUILD_URL}|${env.JOB_NAME} [${env.BUILD_NUMBER}]>") )

                        sh 'scripts/build/load.sh'
                        script {
                            def newCommitFiles = findFiles(glob: 'newcommits*.txt')
                            for (int i = 0; i < newCommitFiles.size(); ++i) {
                                env.NEWCOMMITS = readFile(newCommitFiles[i].path)
                                slackSend (color: '#00FF00', message: "Commits from <${env.BUILD_URL}|${env.JOB_NAME} [${env.BUILD_NUMBER}]>:\n ${env.NEWCOMMITS}" )   
                            }
                        } 
                    }
                }
                stage('Package image') {
                    when { expression {
                            env.BRANCH_NAME.toString().equals('master')
                        }
                    }
                    steps {
                        sh 'scripts/build/pack_image.sh'
                        echo currentBuild.toString()
                        echo currentBuild.result
                    }
                }

                stage('Save with GtWorld') {
                    when { expression {
                            (currentBuild.result == null || currentBuild.result == 'SUCCESS') && env.BRANCH_NAME.toString().equals('master')
                        }
                    }
                    steps {
                        sh 'scripts/build/open_gt_world.sh'
                    }
                }

                stage('Prepare deploy packages') {

                    when {
                        expression {
                            (currentBuild.result == null || currentBuild.result == 'SUCCESS') && env.BRANCH_NAME.toString().equals('master')
                        }
                    }
                    steps {
                        sh 'scripts/build/package.sh'
                        stash includes: 'tagname.txt' , name: 'release_prediction'
                        stash includes: 'GlamorousToolkitWin64*.zip', name: 'winbuild'
                        stash includes: 'lib*.zip', name: 'alllibs'
                        stash includes: 'GT.zip', name: 'gtimage'
                        
                    }
                }

                stage('Upload prerelease') {

                    when {
                        expression {
                            (currentBuild.result == null || currentBuild.result == 'SUCCESS') && env.BRANCH_NAME.toString().equals('master')
                        }
                    }
                    steps {
                        script {
                            withCredentials([sshUserPrivateKey(credentialsId: '31ee68a9-4d6c-48f3-9769-a2b8b50452b0', keyFileVariable: 'identity', passphraseVariable: '', usernameVariable: 'userName')]) {
                                def remote = [:]
                                remote.name = 'deploy'
                                remote.host = 'ip-172-31-37-111.eu-central-1.compute.internal'
                                remote.user = userName
                                remote.identityFile = identity
                                remote.allowAnyHosts = true
                                sshScript remote: remote, script: "scripts/build/clean-tentative.sh"
                            }
                        }
                        sh 'scripts/build/upload-to-tentative.sh'
                    }
                }
            }
        }
        stage('Run Examples') {
            when { expression {
                   (currentBuild.result == null || currentBuild.result == 'SUCCESS') && env.BRANCH_NAME.toString().equals('master')
                }
            }
            parallel {
                stage('Linux') {
                    agent {
                        label "unix"
                    }
                     stages {
                        stage('Download') {
                             steps {
                                sh 'chmod +x scripts/build/parallelsmoke/*.sh'
                                sh 'scripts/build/parallelsmoke/lnx_1_download.sh'
                             }
                        }
                        stage('Linux Examples') {
                             steps {
                                sh 'scripts/build/parallelsmoke/lnx_2_1_examples.sh'
                                junit '*.xml'
                             } 
                        }
                        stage('Smoke Test') {
                             steps {
                                sh 'scripts/build/parallelsmoke/lnx_2_2_smoke.sh'
                             }
                        }
                    }
                }
                stage ('MacOSX') {
                    agent {
                        label "macosx"
                    }
                    environment {
                        CERT = credentials('devcertificate')
                        SUDO = credentials('sudo')
                        APPLEPASSWORD = credentials('notarizepassword')
                        SIGNING_IDENTITY = 'Developer ID Application: feenk gmbh (77664ZXL29)'
                    } 
                    stages {
                        stage('Download') {
                             steps {
                                sh 'echo "${SUDO}" | sudo -S git clean -fdx'
                                sh 'chmod +x scripts/build/parallelsmoke/*.sh'
                                sh 'scripts/build/parallelsmoke/osx_1_download.sh'
                             }
                        }
                        stage('MacOSX Examples') {
                             steps {
                                retry(5) {
                                    sshagent([]) {
                                        sh 'scripts/build/parallelsmoke/osx_2_smoke.sh'
                                        sh 'rm -rf GToolkit-Releaser-*.xml'
                                        junit '*.xml'
                                    }
                                }
                             }
                        }
                        stage('Codesign and Notarize') {
                            when {
                                expression {
                                    (currentBuild.result == null || currentBuild.result == 'SUCCESS')
                                }
                            }
                            steps {
                                sh 'scripts/build/parallelsmoke/osx_3_sign_notarize.sh'

                            }
                        }
                        stage('Upload') {
                            when {
                                expression {
                                    (currentBuild.result == null || currentBuild.result == 'SUCCESS')
                                }
                            }
                             steps {
                                sh 'scripts/build/parallelsmoke/osx_4_upload.sh'
                             }
                        }
                    }
                }
                stage('Windows') {
                    agent {
                        label "windows"
                    }
                    stages {
                        stage('Cleanup') {
                             steps {
                                powershell './scripts/build/parallelsmoke/win_1_cleanup.ps1'
                             }
                        }
                        stage('Download') {
                             steps {
                                powershell './scripts/build/parallelsmoke/win_2_download.ps1'
                             }
                        }
                        stage('Unpack') {
                             steps {
                                powershell './scripts/build/parallelsmoke/win_3_unpack.ps1'
                             }
                        }

                        stage('Windows Examples') {
                             steps {
                                powershell './scripts/build/parallelsmoke/win_4_timeout_examples.ps1'
                                junit '*.xml'
                             }
                        }

                    }
                }
            }
        }
        stage('Deploy release') {
            agent {
                label "unix"
            }
            when { expression {
                    (currentBuild.result == null || currentBuild.result == 'SUCCESS') && env.BRANCH_NAME.toString().equals('master')
                }
            }
            steps {
                dir(MASTER_WORKSPACE) {
                    sh 'chmod +x scripts/build/*.sh'
                    unstash 'release_prediction'
                    unstash 'winbuild'
                    unstash 'alllibs'
                    unstash 'gtimage'  
                    sh 'scripts/build/runreleaser.sh' 
                    sh 'scripts/build/upload.sh'
                    script {
                        TAG_NAME = readFile('tagname.txt').trim()
                        withCredentials([sshUserPrivateKey(credentialsId: '31ee68a9-4d6c-48f3-9769-a2b8b50452b0', keyFileVariable: 'identity', passphraseVariable: '', usernameVariable: 'userName')]) {
                                def remote = [:]
                                remote.name = 'deploy'
                                remote.host = 'ec2-18-197-145-81.eu-central-1.compute.amazonaws.com'
                                remote.user = userName
                                remote.identityFile = identity
                                remote.allowAnyHosts = true
                                sshScript remote: remote, script: "scripts/build/update-latest-links.sh"
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                tsum = getTestSummary()
                slackSend (color: '#00FF00', message: "Successful <${env.BUILD_URL}|${env.JOB_NAME} [${env.BUILD_NUMBER}]>\n$tsum" )
            }
        }

        failure {
            slackSend (color: '#FF0000', message: "Failed  <${env.BUILD_URL}/consoleFull|${env.JOB_NAME} [${env.BUILD_NUMBER}]>")
        }

        unstable {
            script {
                tfailed = getFailedTests()
                tsum = getTestSummary()
                slackSend (color: '#FFFF00', message:  "Unstable <${env.BUILD_URL}/testReport|${env.JOB_NAME} [${env.BUILD_NUMBER}]>\nTest Summary: $tsum\n$tfailed")
            }
        }
    }
}
