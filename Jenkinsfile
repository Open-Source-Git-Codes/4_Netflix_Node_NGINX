pipeline {
    agent any

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    environment {
        SONAR_HOME = tool 'Sonar'
        DOCKER_IMAGE_NAME = 'akkivats777/netflix' // Replace with your DockerHub repository name
    }

    stages {

        stage('Clean Workspace'){
            steps{
                cleanWs()
            }
        }

        stage('Git Checkout') {
            steps {
                git credentialsId: 'git-token', url: 'https://github.com/Open-Source-Git-Codes/4_Netflix_Node_NGINX.git', branch: 'main'
            }
        }

        // stage('Code Compile') {
        //     steps {
        //         sh "mvn clean compile"
        //     }
        // }

        // stage('Unit Testing') {
        //     steps {
        //         sh "mvn test -Dmaven.test.skip=true"
        //     }
        // }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh ''' 
                    $SONAR_HOME/bin/sonar-scanner \
                    -Dsonar.projectKey=netflix \
                    -Dsonar.projectName=netflix \
                    -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage("Quality Gate Check"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonarqube_Token' 
                }
            } 
        }

        stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: ''' 
                            -o './'
                            -s './'
                            -f 'ALL'
                            --prettyPrint
                            --scan ./ 
                            --disableYarnAudit 
                            --disableNodeAudit''', 
                odcInstallation: 'OWASP'
                
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }


        // stage('Trivy File Scan') {
        //     steps {
        //         sh "trivy fs --format table -o trivy-fs-report.html ."
        //     }
        // }

        // stage('Build Jar File') {
        //     steps {
        //         sh "mvn package -Dmaven.test.skip=true"
        //     }
        // }

        // stage('Deploy to Nexus') {
        //     steps {
        //         withMaven(globalMavenSettingsConfig: 'global', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
        //             sh "mvn deploy -Dmaven.test.skip=true"
        //         }
        //     }
        // }

        stage('Docker Image Build') {
            steps {
                script {
                    def buildNumber = env.BUILD_NUMBER
                    sh "docker build --build-arg TMDB_V3_API_KEY=73e83e2b0be89117554a44cc5f5cce2c -t ${DOCKER_IMAGE_NAME}:${buildNumber} ."
                }
            }
        }

        stage('Docker Image Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        def buildNumber = env.BUILD_NUMBER
                        echo "Build Number: ${buildNumber}"

                         // Ensure DOCKER_IMAGE_NAME is defined and is being passed correctly
                        echo "Docker Image Name: ${DOCKER_IMAGE_NAME}"

                        // Correct substitution in the sh block
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_IMAGE_NAME}:${buildNumber}
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                script {
                    def buildNumber = env.BUILD_NUMBER
                    sh "trivy image --format table -o trivy-image-report.html ${DOCKER_IMAGE_NAME}:${buildNumber}"
                }
            }
        }


        stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "4_Netflix_Node_NGINX.git"
                GIT_USER_NAME = "akkivats777"
                GIT_USER_EMAIL = "aakashsharma8527@gmail.com"
                GIT_ORG_NAME = "Open-Source-Git-Codes"
            }
            steps {
                dir('./manifests') {
                    withCredentials([string(credentialsId: 'git-token_secret', variable: 'GITHUB_TOKEN')]) {
                        sh '''
                            git config user.email "$GIT_USER_EMAIL"
                            git config user.name "$GIT_USER_NAME"

                            # Pull the latest changes from the repository (main branch)
                            # Explicitly use the GitHub token for authentication in the URL
                            git pull https://${GITHUB_TOKEN}@github.com/$GIT_ORG_NAME/$GIT_REPO_NAME main --rebase || git rebase --abort

                            # Update the image tag in the deployment.yaml file
                            sed -i "s|image: akkivats777/netflix:[^ ]*|image: akkivats777/netflix:${BUILD_NUMBER}|g" deployment.yaml
                            echo "Updated Image Tag to akkivats777/netflix:${BUILD_NUMBER}"

                            # Check if there are changes before committing
                            git diff --quiet deployment.yaml || (
                                git add deployment.yaml &&
                                git commit -m "Update deployment Image to version ${BUILD_NUMBER}"
                            )
                            # Push the changes back to GitHub (main branch)
                            git push https://${GITHUB_TOKEN}@github.com/$GIT_ORG_NAME/$GIT_REPO_NAME HEAD:main
                        '''
                    }
                }
            }
        }


        stage('Update ArgoCD') {
            steps {
                script {
                    // Assuming ArgoCD is configured to watch this repository/image
                    echo "Triggering ArgoCD to deploy the latest image tag ${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }
    
        stage('Archive Reports') {
            steps {
                archiveArtifacts artifacts: '**/*.xml, **/*.txt, dependency-check-report.xml, trivyfs.txt, trivyimage.txt, trivy-fs-report.html, trivy-image-report.html', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'
                def jenkinsHost = BUILD_URL.split('/job/')[0] // Extract the base Jenkins URL

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                        <h2>${jobName} - Application Build Number - ${buildNumber}</h2>
                        <div style="background-color: ${bannerColor}; padding: 10px;">
                            <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                        </div>
                        <p>Project: ${jobName}</p>
                        <p>Build Number: ${buildNumber}</p>
                        <p>URL: <a href="${BUILD_URL}">${BUILD_URL}</a></p>
                        <p>Check the <a href="${BUILD_URL}console">console output</a>.</p>
                        <p>DNS: <strong>${jenkinsHost}</strong></p>


                        <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                        </div>
                        <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                        </div>
                        <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                            <p style="color: black; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                        </div>
                        

                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} Application - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'aakashsharma8527@gmail.com',
                    from: 'aakashsharma8527@gmail.com',
                    replyTo: 'aakashsharma8527@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html, build.log, dependency-check-report.xml, trivy-fs-report.html, trivy-image-report.html',
                    attachLog: true
                )
            }
        }
    }

}
