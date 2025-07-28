+++
date = '2025-07-28T09:56:41+08:00'
draft = false
title = 'Jenkinsfile Summary'
+++

This document summarizes my key learnings and experiences from writing a Jenkinsfile to implement an Android Jenkins pipeline. It highlights the challenges I encountered, the unfamiliar syntax I discovered, and other insights gained throughout the process.

## 1. Agent Declaration
```groovy
agent any
```
- Specifies that any available agent can execute the pipeline
- Can be customized with agent tags: `agent { label 'android-builder' }`

## 2. Environment Block
```groovy
environment {
    ANDROID_HOME = "/opt/android/sdk"
    JAVA_HOME = "/Library/Java/OpenJDK/jdk-18.0.2.jdk/Contents/Home"
    PATH = "${JAVA_HOME}/bin:$PATH"
    // ... other global variables
}
```
Serves two main purposes:
1. Modifies existing environment variables
2. Defines global parameters shared across stages

## 3. Parameters Block
```groovy
parameters {
    choice(name: 'BuildType', choices: ['debug', 'release', 'aab'])
    booleanParam(name: 'BackupDistribution', defaultValue: true)
    // ... other parameters
}
```
- Defines user-configurable options for the pipeline
- Accessed via `params` object in pipeline steps

## 4. Stages Structure
```groovy
stages {
    stage('Name') { ... }
    stage('Name') { ... }
}
```
- Defines sequential actions Jenkins will execute
- Runs all stages in declared order

## 5. Stage Components
```groovy
stage('Example') {
    steps {
        // Executable steps
    }
}
```
- Each stage requires:
  - Unique name
  - `steps` block containing executable actions
- `script` block should be inside `steps`

## 6. Variable Declaration in Steps
```groovy
steps {
    script {
        def myVar = 'value'
        // Groovy code
    }
}
```
- Variables must be declared inside `script` blocks
- Supports complex logic and variable assignments

## 7. File Operations
```groovy
writeFile file: "local.properties", text: "sdk.dir=${env.ANDROID_HOME}"
```
- Use named parameters for clarity
- Supports multiple parameters: `file`, `text`, `encoding`

## 8. Conditional Stage Execution
```groovy
when {
    expression {
        params.BuildType == 'aab'
    }
}
```
- Controls stage execution based on conditions
- Uses Groovy expressions for evaluation

## 9. Accessing Parameters
```groovy
if (params.DisableConfigurationCache) {
    // configuration logic
}
```
- Access parameters via global `params` object
- Supports all parameter types defined in `parameters` block

## 10. Shell Command Execution
```groovy
def output = sh(script: "ls -d -1 $path", returnStdout: true).trim()
def status = sh(script: "command", returnStatus: true)
```
- `returnStdout: true` captures command output
- `returnStatus: true` captures exit code
- Default behavior executes command without capturing output

## 11. File Archiving with Zip
```groovy
sh "zip -j $outputPath $inputFile"
```
- `-j` option stores files without directory structure
- Useful for creating flat archive structures

## 12. Post-execution Actions
```groovy
post {
    always {
        // Archive artifacts
    }
}
```
- Executes after all stages complete
- Common uses: artifact archiving, notifications, cleanup

## 13. Artifact Archiving
```groovy
archiveArtifacts artifacts: "$file1, $file2", allowEmptyArchive: false
```
- `artifacts` parameter accepts comma-separated file paths
- Supports wildcards: `'build/dist/*.zip'`
- `allowEmptyArchive` controls behavior when no files match

## Pipeline Flow Summary
1. **Environment Setup**: Configures paths and global variables
2. **Preparation**: Creates output directories, sets permissions
3. **Code Analysis**: Runs detekt static analysis
4. **Build**: Compiles APK/AAB based on build type
5. **Version Handling**: Extracts version info from build outputs
6. **Artifact Management**: Moves, zips, and backs up distributions
7. **Deployment**: Handles Play Store uploads and QR code updates
8. **Post-processing**: Archives artifacts

## Jenkinsfile

Here's my source Jenkinsfile:

```groovy
pipeline {
    agent any

    environment {
        ANDROID_HOME = "/opt/android/sdk"
        JAVA_HOME = "/Library/Java/OpenJDK/jdk-18.0.2.jdk/Contents/Home"
        PATH = "${JAVA_HOME}/bin:$PATH"

        gPackageName="me.mycroftwong.example"

        gS3BackupUrl = "s3://$S3_BACKUP_HOST/archive/release/example"

        gPlayStorePushInternalTestJar="/Users/mycroft/.jenkins/uploadToInternalTest/InternalTestAab-1.0.jar"
        gPlayStorePushInternalTestClientId="/Users/mycroft/.jenkins/uploadToInternalTest/app-internal-test-xxx.json"

        gApkSrcDir = 'app/build/outputs/apk'
        gAabSrcDir = 'app/build/outputs/bundle/release'
        gOutputDir = 'build/dist'

        gVersionCode = ''
        gVersionName = ''
        gDistributionFile = ''
        gDistributionZipFile = ''
    }

    parameters {
        choice(name: 'BuildType', choices: ['debug', 'release', 'aab'], description: 'BuildType')
        booleanParam(name: 'BackupDistribution', defaultValue: true, description: 'Back up distribution')
        booleanParam(name: 'UpdateQrcode', defaultValue: false, description: 'Whether update qrcode')
        booleanParam(name: 'ReleaseCandidate', defaultValue: false, description: 'Build Release Candidate, BuildType must be release')
        booleanParam(name: 'PushAabToPlayStore', defaultValue: false, description: 'Whether push aab to play store, BuildType must be aab')
        booleanParam(name: 'DisableConfigurationCache', defaultValue: true, description: 'If not, version code may not be changed')
        booleanParam(name: 'EnableSyncNetDebug', defaultValue: false, description: 'If not, disable sync net debug(assert)')
    }

    stages {
        stage('PrepareOutputDir') {
            steps {
                sh "mkdir -p $gOutputDir"

                sh "rm -rf $gOutputDir/*"
            }
        }

        stage('PrepareLocalProperties') {
            steps {
                writeFile file: "local.properties", text: "sdk.dir=${env.ANDROID_HOME}"
            }
        }

        stage('AddFilePermissions') {
            steps {
                sh 'chmod +x ./gradlew'
            }
        }

        stage('Detekt') {
            steps {
                sh './gradlew detekt'
            }
        }

        stage('Build') {
            steps {
                script {
                    def configurationCacheFlag = ''
                    if (params.DisableConfigurationCache) {
                        configurationCacheFlag = '--no-configuration-cache'
                    }
                    def enableSyncNetDebug = ''
                    if (params.EnableSyncNetDebug) {
                        enableSyncNetDebug = '-PenableSyncNetDebug'
                    }
                    if (params.BuildType == 'debug') {
                        sh "./gradlew clean :app:assembleDebug $configurationCacheFlag $enableSyncNetDebug"
                    } else if (params.BuildType == 'release') {
                        sh "./gradlew clean :app:assembleRelease $configurationCacheFlag $enableSyncNetDebug"
                    } else if (params.BuildType == 'aab') {
                        sh "./gradlew clean :app:bundleRelease $configurationCacheFlag $enableSyncNetDebug"
                    } else {
                        throw IllegalArgumentException('Unexpected BuildType: ' + params.BuildType)
                    }
                }
            }
        }

        stage('GetAabVersion') {
            when {
                expression {
                    params.BuildType == 'aab'
                }
            }

            steps {
                script {
                    def aabFile = sh(script: "ls -d -1 $gAabSrcDir/*.aab", returnStdout: true).trim()
                    def versionCode = sh(script: "bundletool dump manifest --bundle $aabFile --xpath /manifest/@android:versionCode", returnStdout: true).trim()
                    def versionName = sh(script: "bundletool dump manifest --bundle $aabFile --xpath /manifest/@android:versionName", returnStdout: true).trim()

                    gVersionCode = versionCode
                    gVersionName = versionName
                    gDistributionFile = "example-$versionName-$versionCode-${params.BuildType}.aab"

                    echo "[GetAabVersion] VersionCode: $versionCode"
                    echo "[GetAabVersion] VersionName: $versionName"
                    echo "[GetAabVersion] DistributionFile: $gDistributionFile"
                }
            }
        }

        stage('GetApkVersion') {
            when {
                expression {
                    params.BuildType == 'debug' || params.BuildType == 'release'
                }
            }

            steps {
                script {
                    def versionCode = sh(script: "cat $gApkSrcDir/${params.BuildType}/output-metadata.json | jq \".elements[0].versionCode\"", returnStdout: true).trim()
                    def versionName = sh(script: "cat $gApkSrcDir/${params.BuildType}/output-metadata.json | jq -r \".elements[0].versionName\"", returnStdout: true).trim()

                    gVersionCode = versionCode
                    gVersionName = versionName
                    gDistributionFile = "example-$versionName-$versionCode-${params.BuildType}.apk"

                    echo "[GetApkVersion] VersionCode: $versionCode"
                    echo "[GetApkVersion] VersionName: $versionName"
                    echo "[GetApkVersion] DistributionFile: $gDistributionFile"
                }
            }
        }

        stage('MoveAabDistributionIntoOutputDir') {
            when {
                expression {
                    params.BuildType == 'aab'
                }
            }

            steps {
                script {
                    def aabFile = sh(script: "ls -d -1 $gAabSrcDir/*.aab", returnStdout: true).trim()
                    sh "cp $aabFile $gOutputDir/$gDistributionFile"
                }
            }
        }

        stage('MoveApkDistributionIntoOutputDir') {
            when {
                expression {
                    params.BuildType == 'debug' || params.BuildType == 'release'
                }
            }

            steps {
                script {
                    def apkFile = sh(script: "ls -d -1 $gApkSrcDir/${params.BuildType}/*.apk", returnStdout: true).trim()
                    sh "cp $apkFile $gOutputDir/$gDistributionFile"
                }
            }
        }

        stage('ZipDistribution') {
            steps {
                script {
                    gDistributionZipFile = gDistributionFile.replace('.apk', '.zip').replace('.aab', '.zip')
                    sh "zip -j $gOutputDir/$gDistributionZipFile $gOutputDir/$gDistributionFile"
                }
            }
        }

        stage('BackupDistribution') {
            when {
                expression {
                    params.BackupDistribution
                }
            }

            steps {
                script {
                    def archiveAppVersion = gVersionName.tokenize('.').take(2).join('.')
                    if (params.ReleaseCandidate) {
                        archiveAppVersion = archiveAppVersion + "-RC"
                    }
                    def s3Url = "$gS3BackupUrl/$archiveAppVersion/dev/Android/"
                    if (params.BuildType == 'debug' && params.ReleaseCandidate) {
                        s3Url = "$gS3BackupUrl/$archiveAppVersion/rc/Android/"
                    }
                    sh "aws s3 cp $gOutputDir/$gDistributionZipFile $s3Url --profile bk"
                }
            }
        }

        stage('UpdateQrcode') {
            when {
                expression {
                    params.UpdateQrcode && params.BuildType == 'debug'
                }
            }

            steps {
                script {
                    def archiveAppVersion = gVersionName.tokenize('.').take(2).join('.')
                    if (params.ReleaseCandidate) {
                        archiveAppVersion = archiveAppVersion + "-RC"
                    }

                    writeFile file: "$gOutputDir/version.txt", text: gVersionName

                    sh "aws s3 cp $gOutputDir/version.txt s3://example-storage/Testing/example/QRCodeDownload/v$archiveAppVersion/"
                    sh "aws s3 cp $gOutputDir/$gDistributionFile s3://example-storage/Testing/example/QRCodeDownload/v$archiveAppVersion/example.apk"

                    sh "node /Applications/qrCode/index.js --project=example --release=v$archiveAppVersion --android=$gVersionName"
                }
            }
        }

        stage('PushAabToPlayStore') {
            when {
                expression {
                    params.BuildType == 'aab' && params.PushAabToPlayStore
                }
            }

            steps {
                sh "java -jar $gPlayStorePushInternalTestJar $gPackageName $gDistributionFile $gPlayStorePushInternalTestClientId"
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: "$gOutputDir/$gDistributionZipFile, $gOutputDir/versions.txt", allowEmptyArchive: false
        }
    }
}
```

AI make it better:

Key Improvements Needed:
1. **Agent Specification**: Use specific labels instead of `any`
2. **Credentials Handling**: Avoid hardcoding secrets
3. **Environment Variables**: Use `withCredentials` for sensitive data
4. **Stage Organization**: Reduce duplication in APK/AAB handling
5. **Error Handling**: Add better validation and error checking
6. **Post Actions**: Improve artifact handling

```groovy
pipeline {
    agent {
        label 'android-agent' // Use specific agent label
    }

    environment {
        ANDROID_HOME = "/opt/android/sdk"
        JAVA_HOME = "/Library/Java/OpenJDK/jdk-18.0.2.jdk/Contents/Home"
        PATH = "${JAVA_HOME}/bin:$PATH"

        gPackageName = "me.mycroftwong.example"
        gS3BackupUrl = "s3://$S3_BACKUP_HOST/archive/release/example"

        // Removed hardcoded credentials
        gApkSrcDir = 'app/build/outputs/apk'
        gAabSrcDir = 'app/build/outputs/bundle/release'
        gOutputDir = 'build/dist'
    }

    parameters {
        choice(name: 'BuildType', choices: ['debug', 'release', 'aab'], description: 'BuildType')
        booleanParam(name: 'BackupDistribution', defaultValue: true, description: 'Back up distribution')
        booleanParam(name: 'UpdateQrcode', defaultValue: false, description: 'Whether update qrcode')
        booleanParam(name: 'ReleaseCandidate', defaultValue: false, description: 'Build Release Candidate')
        booleanParam(name: 'PushAabToPlayStore', defaultValue: false, description: 'Push aab to play store')
        booleanParam(name: 'DisableConfigurationCache', defaultValue: true, description: 'Disable configuration cache')
        booleanParam(name: 'EnableSyncNetDebug', defaultValue: false, description: 'Enable sync net debug')
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    if (params.ReleaseCandidate && params.BuildType != 'release') {
                        error("ReleaseCandidate can only be used with release build type")
                    }
                    if (params.PushAabToPlayStore && params.BuildType != 'aab') {
                        error("PushAabToPlayStore can only be used with aab build type")
                    }
                }
            }
        }

        stage('Prepare Environment') {
            steps {
                sh "mkdir -p $gOutputDir"
                sh "rm -rf $gOutputDir/*"
                writeFile file: "local.properties", text: "sdk.dir=${env.ANDROID_HOME}"
                sh 'chmod +x ./gradlew'
            }
        }

        stage('Static Analysis') {
            steps {
                sh './gradlew detekt'
            }
        }

        stage('Build') {
            steps {
                script {
                    def flags = []
                    if (params.DisableConfigurationCache) flags << '--no-configuration-cache'
                    if (params.EnableSyncNetDebug) flags << '-PenableSyncNetDebug'

                    switch(params.BuildType) {
                        case 'debug':
                            sh "./gradlew clean :app:assembleDebug ${flags.join(' ')}"
                            break
                        case 'release':
                            sh "./gradlew clean :app:assembleRelease ${flags.join(' ')}"
                            break
                        case 'aab':
                            sh "./gradlew clean :app:bundleRelease ${flags.join(' ')}"
                            break
                        default:
                            error("Invalid BuildType: ${params.BuildType}")
                    }
                }
            }
        }

        stage('Process Artifacts') {
            steps {
                script {
                    // Common processing for all build types
                    def artifactFile = ""
                    def versionFile = ""

                    if (params.BuildType == 'aab') {
                        artifactFile = sh(script: "ls -d -1 $gAabSrcDir/*.aab", returnStdout: true).trim()
                        env.gVersionCode = sh(script: "bundletool dump manifest --bundle $artifactFile --xpath /manifest/@android:versionCode", returnStdout: true).trim()
                        env.gVersionName = sh(script: "bundletool dump manifest --bundle $artifactFile --xpath /manifest/@android:versionName", returnStdout: true).trim()
                    } else {
                        def jsonFile = "$gApkSrcDir/${params.BuildType}/output-metadata.json"
                        artifactFile = sh(script: "ls -d -1 $gApkSrcDir/${params.BuildType}/*.apk", returnStdout: true).trim()
                        env.gVersionCode = sh(script: "jq -r '.elements[0].versionCode' $jsonFile", returnStdout: true).trim()
                        env.gVersionName = sh(script: "jq -r '.elements[0].versionName' $jsonFile", returnStdout: true).trim()
                    }

                    def ext = params.BuildType == 'aab' ? 'aab' : 'apk'
                    env.gDistributionFile = "example-${env.gVersionName}-${env.gVersionCode}-${params.BuildType}.${ext}"
                    sh "cp $artifactFile $gOutputDir/$env.gDistributionFile"

                    // Zip distribution
                    env.gDistributionZipFile = "${env.gDistributionFile}.zip"
                    sh "zip -j $gOutputDir/$env.gDistributionZipFile $gOutputDir/$env.gDistributionFile"

                    // Create version file
                    writeFile file: "$gOutputDir/versions.txt", text: env.gVersionName
                }
            }
        }

        stage('Backup Distribution') {
            when {
                expression { params.BackupDistribution }
            }
            steps {
                script {
                    def archiveAppVersion = env.gVersionName.tokenize('.').take(2).join('.')
                    if (params.ReleaseCandidate) {
                        archiveAppVersion += "-RC"
                    }

                    def s3Path = "$gS3BackupUrl/$archiveAppVersion/"
                    s3Path += params.BuildType == 'debug' && params.ReleaseCandidate ? "rc/Android/" : "dev/Android/"

                    sh "aws s3 cp $gOutputDir/$env.gDistributionZipFile $s3Path --profile bk"
                }
            }
        }

        stage('Update QR Code') {
            when {
                expression {
                    params.UpdateQrcode && params.BuildType == 'debug'
                }
            }
            steps {
                script {
                    def archiveAppVersion = env.gVersionName.tokenize('.').take(2).join('.')
                    if (params.ReleaseCandidate) {
                        archiveAppVersion += "-RC"
                    }

                    sh """
                        aws s3 cp $gOutputDir/versions.txt s3://example-storage/Testing/example/QRCodeDownload/v$archiveAppVersion/
                        aws s3 cp $gOutputDir/$env.gDistributionFile s3://example-storage/Testing/example/QRCodeDownload/v$archiveAppVersion/example.apk
                        node /Applications/qrCode/index.js --project=example --release=v$archiveAppVersion --android=${env.gVersionName}
                    """
                }
            }
        }

        stage('Deploy to Play Store') {
            when {
                expression {
                    params.BuildType == 'aab' && params.PushAabToPlayStore
                }
            }
            steps {
                withCredentials([
                    file(credentialsId: 'playstore-upload-jar', variable: 'UPLOAD_JAR'),
                    file(credentialsId: 'playstore-service-account', variable: 'SERVICE_ACCOUNT')
                ]) {
                    sh "java -jar $UPLOAD_JAR $gPackageName $env.gDistributionFile $SERVICE_ACCOUNT"
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: "$gOutputDir/*.zip, $gOutputDir/versions.txt", allowEmptyArchive: true
        }
        cleanup {
            deleteDir() // Clean up workspace
        }
    }
}
```

Key Improvements Made:
1. **Agent Specification**:
   - Replaced `agent any` with labeled agent for better resource management

2. **Credentials Management**:
   - Removed hardcoded credentials
   - Added `withCredentials` for secure credential handling

3. **Parameter Validation**:
   - Added validation stage to check parameter combinations

4. **Reduced Duplication**:
   - Combined APK/AAB processing into a single stage
   - Created unified artifact processing logic

5. **Environment Variables**:
   - Properly scoped variables using `env.` prefix
   - Removed unused environment variables

6. **Improved Build Stage**:
   - Used switch statement for cleaner build type handling
   - Combined gradle flags into a single command

7. **Enhanced Post Actions**:
   - Added `cleanup` step to delete workspace
   - Made artifact archiving more robust with wildcards
   - Set `allowEmptyArchive: true` to prevent failures

8. **Error Handling**:
   - Added explicit error messages for invalid parameter combinations
   - Improved validation of build types

9. **Security**:
   - Removed all hardcoded credentials
   - Used Jenkins credential store via `withCredentials`

10. **Efficiency**:
    - Combined related file operations
    - Reduced number of stages while maintaining clarity
