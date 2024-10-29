/**
 * This Jenkins script is to build Kotlin Multiplatform project
 **/

pipeline {
    
    agent any
    
    environment {
        // Declare global environment variables
        // General android project ex.: 'app/build/outputs/apk/debug/*.apk'
        // Kotlin Multiplatform project ex.: 'composeApp/build/outputs/apk/debug/*.apk'
        BUILD_PATH = 'composeApp/build/outputs/apk'
        BUILD_APK_PATH = ''
        DESTINATION_PATH = '/Users/touhid/Desktop/test' // Just to get apk easily
        
        MY_PROPERTIES_FILE = "local.properties"
        GRADLE_FILE = 'composeApp/build.gradle.kts'  // Path to your Gradle file
        
        PROJECT_GIT_URL = 'https://github.com/TouhidApps/Unit-Test-Kotlin-Multiplatform-Compose.git'

        APP_ID = '1:1050128836546:android:319e12c85411a6f08fd85b'  // Replace with your Firebase app ID
        FIREBASE_SERVICE_AC_KEY_PATH = '/Users/touhid/Downloads/totax-2b5f9-firebase-adminsdk-6iuqx-bf59c84cf0.json'  // Replace with your file path
        
    }
    
    stages {
    
        stage('Store Value') {
            steps {
                script {
                    // Store the parameter value for future builds
                    //    writeFile file: MY_PROPERTIES_FILE, text: "VERSION_NAME=${params.VERSION_NAME}"
                    
                    // Read the properties file
                    def fileContent = readFile("${MY_PROPERTIES_FILE}").trim().split('\n').collectEntries { line ->
                        def (key, value) = line.split('=')
                        [(key.trim()): value.trim()]
                    }
                    
                    // Log the current file content
                    echo "File Content:\n${fileContent}"
                    
                    // Update the properties with the new value
                    if (params.VERSION_CODE) {
                        fileContent['VERSION_CODE'] = params.VERSION_CODE
                    }
                    if (params.VERSION_NAME) {
                        fileContent['VERSION_NAME'] = params.VERSION_NAME
                    }
                    if (params.RELEASE_NOTES) {
                        fileContent['RELEASE_NOTES'] = params.RELEASE_NOTES.replaceAll('\n', '\t') // Replace newlines with tabs, not to inturrupt new line of other key values values
                    }




                    
                    // Write back to the properties file
                    writeFile file: "${MY_PROPERTIES_FILE}", text: fileContent.collect { "${it.key}=${it.value}" }.join('\n')
                    
                }
            }
        }
        
        stage('Input Parameter') { // Keep this stage after 'Store Value' so after storing it can load new values
            steps {
                script {
                    // Define a parameter dynamically based on the environment variable
                    def fileContent = readFile("${MY_PROPERTIES_FILE}").trim().split('\n').collectEntries { line ->
                        def (key, value) = line.split('=')
                        [(key.trim()): value.trim()]
                    }
                    echo "${fileContent['VERSION_CODE']}"
                    echo "${fileContent['VERSION_NAME']}"
                    echo "${fileContent['RELEASE_NOTES']}"
                    properties([
                        parameters([
                            choice(name: 'BUILD_TYPE', choices: ['debug', 'release'], description: 'Select build type'),
                            string(name: 'FIREBASE_TESTERS', defaultValue: 'uat-testers,uat-testers', description: 'Enter Firebase tester group (comma-separated)'),
                            string(name: 'VERSION_CODE', defaultValue: "${fileContent['VERSION_CODE']}", description: "Increase app version code (Current: ${fileContent['VERSION_CODE']})"),
                            string(name: 'VERSION_NAME', defaultValue: "${fileContent['VERSION_NAME']}", description: "Increase app version name (Current: ${fileContent['VERSION_NAME']})"),
                            text(name: 'RELEASE_NOTES', defaultValue: "${fileContent['RELEASE_NOTES'].replaceAll('\t', '\n')}", description: 'Release notes:')
                        ])
                    ])
                }
            }
        }
        
        stage('Checkout') {
            steps {
                script {
                    // Check out a specific branch
                    git branch: 'main', url: "${PROJECT_GIT_URL}"
                }
            }
        }
        
        stage('Preparation') {
            steps {
                script {
                    
                    // Define Groovy variables
                    def buildType = params.BUILD_TYPE
                    def buildPath = "${BUILD_PATH}/${buildType}"
                    BUILD_APK_PATH = "${buildPath}/*.apk"
                    // Make directory if not exists
                    sh "mkdir -p ${buildPath}"
                    sh "mkdir -p ${DESTINATION_PATH}"
                    // Delete previous build if available
                    sh "rm -rf ${buildPath}/*"
                    
                }
            }
        }
        
        stage('Update Version in Gradle File') {
            steps {
                script {
                    // Read the existing gradle.kts file
                    def gradleContent = readFile("${GRADLE_FILE}")

                    // Replace versionCode and versionName
                    gradleContent = gradleContent.replaceAll(/versionCode\s*=\s*\d+/, "versionCode = ${params.VERSION_CODE}")
                    gradleContent = gradleContent.replaceAll(/versionName\s*=\s*\".*?\"/, "versionName = \"${params.VERSION_NAME}\"")

                    // Write back to the gradle.kts file
                    writeFile(file: "${GRADLE_FILE}", text: gradleContent)
                    echo "Updated ${GRADLE_FILE} with versionCode=${params.VERSION_CODE} and versionName=${params.VERSION_NAME}"
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    // or use `gradlew.bat` for Windows
                    def buildCommand = "./gradlew assemble${BUILD_TYPE.capitalize()}"
                    sh buildCommand
                }
            }
        }
        
        stage('Archive') {
            steps {
                archiveArtifacts artifacts: BUILD_APK_PATH, allowEmptyArchive: true
            }
        }
        
        stage('Pause 3 seconds') {
            steps {
                // Pause for 3 seconds to make sure build file has written to storage perfectly
                script {
                    sleep(3)  // Pause for 3 seconds
                }
            }
        }
        
        stage('Copy to Desktop') {
            steps {
                // Copy the APK to a specific location
                script {
                    sh "cp ${BUILD_APK_PATH} ${DESTINATION_PATH}"
                }
            }
        }
        
        stage('Upload to firebase distribution') {
            steps {
                script {
                    // Upload APK to Firebase App Distribution
                    def fileListCmd = "ls ${BUILD_APK_PATH} | head -n 1" //  | head -n 1 = for first file found
                    def apkFilePath = sh(script: fileListCmd, returnStdout: true).trim()
                    
                    echo "Uploading apk file path: ${apkFilePath}"
                    
                    // Authenticate with Firebase
                    env.GOOGLE_APPLICATION_CREDENTIALS = "${FIREBASE_SERVICE_AC_KEY_PATH}"
                    
                    // Upload to Firebase App Distribution
                    def firebaseUploadCmd = """
                    firebase appdistribution:distribute "${apkFilePath}" \
                      --app "${APP_ID}" \
                      --groups "${params.FIREBASE_TESTERS}" \
                      --release-notes "${params.RELEASE_NOTES}"
                      """
                      
                    echo "Executing: ${firebaseUploadCmd}"
                    sh firebaseUploadCmd
                    
                }
            }
        }
     
        // stage('Send notification') {
        //     steps {
        //         script {
        //             if (currentBuild.result == 'SUCCESS') {
        //                 // Send success notification
        //             } else {
        //                 // Send failure notification
        //             }
        //         }
        //     }
        // }
        
        
     //   stage('Test') {
     //       steps {
      //          sh './gradlew test'
     //       }
     //   }
        // post {
        //     always {
        //         mail to: 'your-email@example.com',
        //              subject: "Build ${currentBuild.currentResult}: Job '${env.JOB_NAME}'",
        //              body: "Check console output at ${env.BUILD_URL}"
        //     }
        //  }
    }
    
}
