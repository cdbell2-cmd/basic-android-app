#!/usr/bin/env groovy
// Android MultiBranch Pipeline — BasicApp
// Stage structure mirrors VirgoAndroidMultiBranch_ARM64 (Adobe DCM) on CloudBees CI — dcmvenus controller

def isReleaseBranch      = env.BRANCH_NAME ==~ /release\/.*/
def isDevelopBranch      = env.BRANCH_NAME == 'develop'
def isMainBranch         = env.BRANCH_NAME == 'master'
def isHotfixBranch       = env.BRANCH_NAME ==~ /hotfix\/.*/
def isBetaBranch         = env.BRANCH_NAME ==~ /beta-.*/

def isDistributionBranch = isReleaseBranch || isMainBranch || isHotfixBranch || isBetaBranch || isDevelopBranch
def isPlayStoreBranch    = isReleaseBranch || isMainBranch

pipeline {
    agent {
        // Targets the macOS build agents (dcm-csbm-mac-175..184 on dcmvenus)
        label 'mac'
    }

    options {
        buildDiscarder(logRotator(
            daysToKeepStr:         '7',
            numToKeepStr:          isDevelopBranch ? '10' : '5',
            artifactDaysToKeepStr: '-1',
            artifactNumToKeepStr:  '-1'
        ))
        timeout(time: 2, unit: 'HOURS')
    }

    // HockeyAppToken, dcmbuildAPIToken, androidKeyAlias, androidKeyStorePwd,
    // androidKeyStoreAscii injected by the MultiBranch folder
    // (EnvVarsFolderProperty in multibranch-config.xml)

    environment {
        GRADLE_PROJECT   = 'app/BasicApp'
        APP_MODULE       = 'app'
        OUTPUT_DIR       = "${WORKSPACE}/build/output"
        ANDROID_SDK_ROOT = "${HOME}/Library/Android/sdk"
    }

    stages {

        // ── 1. CHECKOUT_SOURCES ───────────────────────────────────────────────
        // Mirrors VirgoAndroid: checkout + clean Gradle dependency and build caches
        stage('Checkout_Sources') {
            steps {
                checkout scm
                sh '''
                    rm -rf ~/.gradle/caches
                    rm -rf ~/.gradle/build-cache-1
                    mkdir -p "${OUTPUT_DIR}"
                '''
            }
        }

        // ── 2. BUILD_TEST (parallel) ──────────────────────────────────────────
        // Mirrors VirgoAndroid: virgoAndroidBuildStage + androidExecTestStage run in parallel
        stage('Build_Test') {
            parallel {

                stage('Build') {
                    steps {
                        sh """
                            cd "${GRADLE_PROJECT}"
                            ./gradlew ${APP_MODULE}:assembleDebug \
                                --no-daemon \
                                --stacktrace
                            cp ${APP_MODULE}/build/outputs/apk/debug/*.apk "${OUTPUT_DIR}/"
                        """
                    }
                }

                stage('Test') {
                    steps {
                        sh """
                            cd "${GRADLE_PROJECT}"
                            ./gradlew ${APP_MODULE}:testDebugUnitTest \
                                --no-daemon \
                                --stacktrace
                        """
                    }
                    post {
                        always {
                            junit testResults: '**/test-results/**/*.xml', allowEmptyResults: true
                        }
                    }
                }

            }
        }

        // ── 3. LINT_AND_SONAR ─────────────────────────────────────────────────
        // Mirrors VirgoAndroid: Virgo_Test_Lint_Sonar_Validation stage
        stage('Lint_And_Sonar') {
            steps {
                sh """
                    cd "${GRADLE_PROJECT}"
                    ./gradlew ${APP_MODULE}:lint \
                        --no-daemon \
                        --stacktrace
                """
            }
            post {
                always {
                    androidLint pattern: '**/lint-results-*.xml', allowEmpty: true
                }
            }
        }

        // ── 4. POST_TO_ARTIFACTORY ────────────────────────────────────────────
        // Mirrors VirgoAndroid: archive APK to Artifactory after successful build+test
        stage('Post_To_Artifactory') {
            when { expression { isDistributionBranch } }
            steps {
                archiveArtifacts(
                    artifacts:          'build/output/**/*.apk',
                    fingerprint:        true,
                    allowEmptyArchive:  true
                )
                echo "Artifacts posted — branch: ${env.BRANCH_NAME}, build: ${env.BUILD_NUMBER}"
            }
        }

        // ── 5. POST_TO_FIREBASE ───────────────────────────────────────────────
        // Mirrors VirgoAndroid: Firebase App Distribution via HockeyApp
        stage('Post_To_Firebase') {
            when { expression { isDistributionBranch } }
            steps {
                sh """
                    APK=\$(find "${OUTPUT_DIR}" -name '*-debug.apk' | head -1)
                    curl \
                        -F "status=2" \
                        -F "notify=1" \
                        -F "notes=Branch: ${env.BRANCH_NAME} | Build: ${env.BUILD_NUMBER}" \
                        -F "notes_type=0" \
                        -F "apk=@\${APK}" \
                        -H "X-HockeyAppToken: ${env.HockeyAppToken}" \
                        https://rink.hockeyapp.net/api/2/apps/upload
                """
                archiveArtifacts artifacts: 'build/output/**/*.apk', fingerprint: true
            }
        }

        // ── 6. RELEASE_SIGN ───────────────────────────────────────────────────
        // Mirrors VirgoAndroid: APK signing with release keystore credentials
        // Analogous to iOS AppStore_Resign
        stage('Release_Sign') {
            when { expression { isPlayStoreBranch } }
            steps {
                sh """
                    cd "${GRADLE_PROJECT}"
                    ./gradlew ${APP_MODULE}:assembleRelease \
                        --no-daemon \
                        --stacktrace \
                        -PandroidKeyAlias="${env.androidKeyAlias}" \
                        -PandroidKeyStorePwd="${env.androidKeyStorePwd}"
                    cp ${APP_MODULE}/build/outputs/apk/release/*.apk "${OUTPUT_DIR}/"
                """
                archiveArtifacts artifacts: 'build/output/**/*-release*.apk', fingerprint: true
            }
        }

        // ── 7. UPLOAD_TO_PLAYSTORE ────────────────────────────────────────────
        // Mirrors VirgoAndroid: publish release APK; analogous to iOS Upload_To_TestFlight
        stage('Upload_To_PlayStore') {
            when { expression { isPlayStoreBranch } }
            steps {
                sh """
                    APK=\$(find "${OUTPUT_DIR}" -name '*-release*.apk' | head -1)
                    echo "Uploading \${APK} to Play Store"
                    # Replace with Fastlane supply or Google Play Developer API
                """
            }
        }

        // ── 8. GENERATEANDPOST_CSDASHBOARDCSV ─────────────────────────────────
        // Mirrors VirgoAndroid: publish build metrics CSV on develop and release branches
        stage('GenerateAndPost_CSDashboardCSV') {
            when {
                anyOf {
                    expression { isDevelopBranch }
                    expression { isReleaseBranch }
                }
            }
            steps {
                sh '''
                    echo "build_number,branch,status,timestamp" > build/output/dashboard.csv
                    echo "${BUILD_NUMBER},${BRANCH_NAME},SUCCESS,$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> build/output/dashboard.csv
                '''
                archiveArtifacts artifacts: 'build/output/dashboard.csv', fingerprint: true
            }
        }

    } // end stages

    post {
        always {
            // Send_Email — mirrors VirgoAndroid post-build notification
            script {
                def result  = currentBuild.currentResult ?: 'SUCCESS'
                def subject = "[${result}] ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                def body    = "Branch: ${env.BRANCH_NAME}\nBuild: ${env.BUILD_NUMBER}\nStatus: ${result}\nURL: ${env.BUILD_URL}"
                echo "Send_Email\nSubject: ${subject}\n${body}"
            }
            deleteDir()
        }
        failure {
            echo "Build FAILED — branch: ${env.BRANCH_NAME}, build: ${env.BUILD_NUMBER}"
        }
        unstable {
            echo "Build UNSTABLE (test failures) — branch: ${env.BRANCH_NAME}"
        }
    }
}
