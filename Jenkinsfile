#!/usr/bin/env groovy
// Android MultiBranch Pipeline — BasicApp (with CloudBees Workspace Caching)
// Stage structure mirrors VirgoAndroidMultiBranch_ARM64 (Adobe DCM) on CloudBees CI — dcmvenus controller
// Caching added per: android-workspace-caching-readme.md

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
        ANDROID_HOME     = "${HOME}/Library/Android/sdk"
        ANDROID_SDK_ROOT = "${HOME}/Library/Android/sdk"
    }

    stages {

        // ── 1. RESTORE_CACHE ──────────────────────────────────────────────────
        // Restore Gradle dependency and build caches before checkout so they are
        // in place before any work begins.  On first run these are no-ops.
        // Both paths are outside the workspace, so dir() is required —
        // readCache/writeCache do not support absolute paths.
        stage('Restore_Cache') {
            steps {
                dir("${HOME}/.gradle/caches/modules-2") {
                    readCache name: 'gradle-dependencies'
                }
                dir("${HOME}/.gradle/caches/build-cache-1") {
                    readCache name: 'gradle-build-cache'
                }
            }
        }

        // ── 2. CHECKOUT_SOURCES ───────────────────────────────────────────────
        // rm -rf ~/.gradle/caches removed: that was the sole cause of cold builds.
        // Only the file-system journal is cleared here. The journal records absolute
        // paths to workspace files from the previous build; deleteDir() removes those
        // files at the end of each build, making the journal stale. A stale journal
        // causes mergeDebugResources (and similar tasks) to fail with
        // FileNotFoundException when Gradle tries to hash an output it believes exists.
        // modules-2/ (Maven artifacts) and build-cache-1/ (task output cache) do not
        // contain workspace paths and are left intact for the warm build.
        stage('Checkout_Sources') {
            steps {
                checkout scm
                sh '''
                    rm -rf ~/.gradle/caches/journal-1
                    mkdir -p "${OUTPUT_DIR}"
                '''
            }
        }

        // ── 3. BUILD_TEST (parallel) ──────────────────────────────────────────
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

        // ── 4. LINT_AND_SONAR ─────────────────────────────────────────────────
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
                    archiveArtifacts artifacts: '**/lint-results-*.xml', allowEmptyArchive: true
                }
            }
        }

        // ── 5. POST_TO_ARTIFACTORY ────────────────────────────────────────────
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

        // ── 6. POST_TO_FIREBASE ───────────────────────────────────────────────
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

        // ── 7. RELEASE_SIGN ───────────────────────────────────────────────────
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

        // ── 8. UPLOAD_TO_PLAYSTORE ────────────────────────────────────────────
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

        // ── 9. GENERATEANDPOST_CSDASHBOARDCSV ─────────────────────────────────
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
        // ── WRITE CACHES ──────────────────────────────────────────────────────
        // Only on successful, non-PR builds:
        //   - post { success } ensures a failing build never poisons the cache
        //   - !env.CHANGE_ID skips PR builds; they read the target branch cache
        //     via automatic multibranch fallback but must not overwrite it
        success {
            script {
                if (!env.CHANGE_ID) {
                    dir("${HOME}/.gradle/caches/modules-2") {
                        writeCache name: 'gradle-dependencies',
                            includes: '**',
                            excludes: '**/*.lock'
                    }
                    dir("${HOME}/.gradle/caches/build-cache-1") {
                        writeCache name: 'gradle-build-cache', includes: '**'
                    }
                }
            }
        }

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
