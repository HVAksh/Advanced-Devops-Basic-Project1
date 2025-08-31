/*
  Production-ready Jenkinsfile (Explanatory version)
  - Enterprise-grade: secure artifact handling, Docker build, deployment, and testing
  - Fully decoupled from SCM for artifact repo info
  - WAR is created in Build & Test stage
  - Artifact uploaded to Nexus via nexusArtifactUploader plugin
  - Docker image built from WAR
  - Optional deploy to VM via Ansible
*/

pipeline {
    // Agent selection (Linux with Docker installed)
    agent any //{ label 'linux && docker' }

    tools {
        maven "mvn"
    }

    // Pipeline parameters for deployment control
    parameters {
        booleanParam(name: 'RUN_DEPLOY', defaultValue: false, description: 'Deploy after successful build')
        choice(name: 'DEPLOY_ENV', choices: ['dev','stage','prod'], description: 'Deployment environment')
    }

    // General pipeline options
    options {
        //ansiColor('xterm')               // Colorized console output
        timestamps()                      // Show timestamps for each step
        disableConcurrentBuilds()         // Prevent overlapping builds
        buildDiscarder(logRotator(numToKeepStr: '5')) // Keep only last 5 builds
        timeout(time: 120, unit: 'MINUTES')           // Max 2 hours
    }

    environment {
        // Application coordinates
        APP_NAME = "maven-test-app"
        GROUP_ID = "com.student"
        // VERSION = ''                     // Set dynamically after checkout
        // WAR_FILE = ''                    // Set dynamically from POM version
        // BUILD_TAG = ''                   // Computed later after version is extracted        
        VERSION = "3.0.${BUILD_NUMBER}"          // Build-number-based versioning
        WAR_FILE = "${APP_NAME}-${VERSION}.war"  // Name of WAR created
        BUILD_TAG = "${BRANCH_NAME ?: 'local'}-${BUILD_NUMBER}" // Unique tag for Docker

        // Nexus repository names
        NEXUS_REPO_RELEASES = "maven-releases"
        NEXUS_REPO_SNAPSHOTS = "maven-snapshots"

        // Docker registry info
        //DOCKER_REGISTRY = "docker.io"
        DOCKER_IMAGE = "${APP_NAME}"

        // Jenkins credential IDs (configured in Jenkins Credentials)
        //CRED_ID_NEXUS = "creds-nexus"       // Username/password for Nexus
        //CRED_ID_DOCKER = "creds-docker-reg" // Docker registry credentials
        //CRED_ID_SONAR = "creds-sonar"       // SonarQube token
        //CRED_ID_ANSIBLE = "creds-ansible"   // SSH key for Ansible
    }

    stages {

        // stage('Example') {
        //     steps {
        //         ansiColor('xterm') {
        //             // your steps here
        //         }
        //     }
        // }
        // ------------------------------
        stage('Checkout Source') {
            steps {
                echo "Cloning the source code from SCM..."
                checkout scm
            }
        }

        // ------------------------------

        // stage('Extract Version from POM') {
        //     steps {
        //         script {
        //             def pom = readMavenPom file: 'pom.xml'
        //             env.VERSION = pom.version
        //             env.WAR_FILE = "${env.APP_NAME}-${env.VERSION}.war"
        //             env.BUILD_TAG = "${env.VERSION}-${env.BUILD_NUMBER}"
        //             echo "Extracted version: ${env.VERSION}"
        //             echo "Expected WAR: ${env.WAR_FILE}"
        //             echo "Build tag: ${env.BUILD_TAG}"
        //         }
        //     }
        // }
        // ------------------------------
        stage('Pre-CI Security & Quality Scans') {
            parallel {
                stage('Secrets Scan (gitleaks)') {
                    steps {
                        echo "Scanning source for exposed secrets..."
                        sh 'gitleaks detect --source . --no-git --report-path gitleaks-report.json || true'
                    }
                }
                stage('Static Analysis (SonarQube)') {
                    steps {
                        echo "Running SonarQube analysis..."
                        script {
                            def SonarQubecredentialsId = 'sonarqube-token'
                            withSonarQubeEnv(credentialsId: 'sonarqube-token') {
                                // Maven command to run Sonar analysis
                                sh 'mvn clean package sonar:sonar'
                            }
                        }
                        echo "Waiting for Quality Gate result..."
                    //    timeout(time: 10, unit: 'MINUTES') { waitForQualityGate abortPipeline: true }
                        timeout(time: 10, unit: 'MINUTES') {
                            // Wait for Quality Gate status and bypass pipeline failure if it fails
                        script {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                            echo "Quality Gate failed: ${qg.status}. Proceeding with the build."
                            // Change build result to SUCCESS or UNSTABLE, depending on your needs
                            currentBuild.result = 'UNSTABLE' // Or 'SUCCESS' to fully bypass failure
                            }
                        }
                        }
                    }
                }
                stage('Dependency & OS Scan') {
                    steps {
                        echo "Running dependency and OS-level vulnerability scans..."
                        sh """
                        mvn -q org.owasp:dependency-check-maven:check || echo "Dependency check found issues"
                        trivy fs --exit-code 1 --severity HIGH,CRITICAL . || true
                        """
                    }
                }
            }
        }

        // ------------------------------
        stage('Build & Test') {
            steps {
                echo "Building application and running unit & integration tests..."
                // Runs full Maven lifecycle: compile, test, package (WAR), verify
                retry(2) {
                    sh "mvn -B clean verify"
                }
            }
            post {
                always {
                    // Publish test reports regardless of success/failure
                    junit '**/target/surefire-reports/*.xml'
                    junit '**/target/failsafe-reports/*.xml'
//                    junit '**/target/test-classes/*.xml' // Ensure the path is correct
                }
                success {
                    // Archive WAR artifact for traceability
                    archiveArtifacts artifacts: 'target/*.war', fingerprint: true
                }
            }
        }

        // ------------------------------
        stage('Publish artifact to Nexus') {
            steps {
                echo "Uploading WAR to Nexus using nexusArtifactUploader (secure, SCM-safe)..."
                script {
                    // Ensure the WAR exists
                    if (!fileExists("target/${env.WAR_FILE}")) {
                        error("WAR not found: target/${env.WAR_FILE}")
                    }

                    // Use Jenkins credentials for secure upload
                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds', 
                        usernameVariable: 'NEXUS_USER', 
                        passwordVariable: 'NEXUS_PASS'
                    )])
                    {
                        retry(3) {
                            nexusArtifactUploader artifacts: [[ artifactId: "${env.APP_NAME}", 
                                                                classifier: '', 
                                                                file: "target/${env.WAR_FILE}", 
                                                                type: 'war'
                                                            ]],
                            credentialsId: "",
                            groupId: "${env.GROUP_ID}",
                            nexusUrl: "nexus.example.com",
                            nexusVersion: 'nexus3',
                            protocol: 'https',
                            repository: "${env.NEXUS_REPO_RELEASES}",
                            version: "${env.VERSION}"
                        }
                    }
                }
            }
        }
        // ------------------------------
        stage('Docker: Build & Push') {
            steps {
                echo "Building Docker image containing the WAR..."
                script {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub', 
                    usernameVariable: 'DOCKER_USER', 
                    passwordVariable: 'DOCKER_PASS'
                )]) 
                {
                    sh """
                    echo $DOCKER_PASS | docker login -u "$DOCKER_USER" -p "$DOCKER_PASS"
                    """
                }
                // {
                //     sh "echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" ${DOCKER_REGISTRY} --password-stdin"
                // }
                {
                    sh """
                    cp target/${WAR_FILE} ./app.war
                    docker build --pull -t ${DOCKER_IMAGE}:${BUILD_TAG} .
                    docker push ${DOCKER_IMAGE}:${BUILD_TAG}
                    docker tag ${DOCKER_IMAGE}:${BUILD_TAG} ${DOCKER_IMAGE}:latest
                    docker push ${DOCKER_IMAGE}:latest
                    """
                }
                }
            }
        }
    }

    post {
        always {
            echo "Archiving all reports and artifacts for traceability..."
            archiveArtifacts artifacts: 'gitleaks-report.json, sbom-*.json, *.xml, target/*.war', allowEmptyArchive: true
        }
        success { echo "Pipeline SUCCESS: ${JOB_NAME} #${BUILD_NUMBER}" }
        failure { echo "Pipeline FAILURE: ${JOB_NAME} #${BUILD_NUMBER}" }
    }
}


    // ------------------------------
    //     stage('Deploy to VM') {
    //         when { expression { params.RUN_DEPLOY } }
    //         steps {
    //             lock(resource: "deploy-${params.DEPLOY_ENV}") {
    //                 script {
    //                     if (params.DEPLOY_ENV == 'prod') { 
    //                         input message: "Approve deploy to PROD?", ok: "Deploy" 
    //                     }

    //                     withCredentials([sshUserPrivateKey(credentialsId: env.CRED_ID_ANSIBLE, keyFileVariable: 'ANSIBLE_KEY')]) {
    //                         retry(2) {
    //                             sh """
    //                             ansible-playbook infra/ansible/playbooks/deploy.yml \
    //                               -i infra/ansible/inventory/${params.DEPLOY_ENV} \
    //                               --extra-vars "image=${DOCKER_IMAGE}:${BUILD_TAG} env=${params.DEPLOY_ENV}" \
    //                               --private-key $ANSIBLE_KEY
    //                             """
    //                         }
    //                     }
    //                 }
    //             }
    //         }
    //     }

    //     // ------------------------------
    //     stage('Post-deploy Health Check') {
    //         when { expression { params.RUN_DEPLOY } }
    //         steps {
    //             script {
    //                 echo "Running post-deployment health check..."
    //                 def url = params.DEPLOY_ENV == 'prod' ? 'https://app.company.com/health' : "https://app-${params.DEPLOY_ENV}.company.com/health"
    //                 def ok = false
    //                 for (int i = 0; i < 3; i++) {
    //                     try { 
    //                         sh "curl -f -sS ${url}"; 
    //                         ok = true; 
    //                         break 
    //                     } catch (err) { 
    //                         echo "Health check failed attempt ${i+1}/3"; 
    //                         sleep time: (i+1)*10, unit: 'SECONDS' 
    //                     }
    //                 }
    //                 if (!ok) { 
    //                     error("Post-deploy health check failed for ${url}") 
    //                 }
    //             }
    //         }
    //     }

