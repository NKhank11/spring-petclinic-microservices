def VALID_SERVICES = [
    'spring-petclinic-admin-server',
    'spring-petclinic-api-gateway',
    'spring-petclinic-config-server',
    'spring-petclinic-genai-service',
    'spring-petclinic-customers-service',
    'spring-petclinic-discovery-server',
    'spring-petclinic-vets-service',
    'spring-petclinic-visits-service',
]

def AFFECTED_SERVICES = ''

pipeline {
    agent any
    tools {
        maven 'Maven-3.9.4'
        jdk 'OpenJDK-17'
    }
    environment {
        DOCKER_REGISTRY = 'nkhank11'
        GITHUB_CREDENTIALS_ID = 'github_pat'
        GKE_CREDENTIALS_ID = 'gke_credentials'
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub_credentials'
        GITHUB_REPO_URL = 'https://github.com/NKhank11/spring-petclinic-microservices.git'
        MANIFEST_REPO = 'github.com/NKhank11/petclinic-gitops.git'
    }
    stages {
        stage('Clone Code') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: '**']],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [
                            [$class: 'CloneOption', noTags: false, depth: 0, shallow: false],
                            [$class: 'PruneStaleBranch']
                        ],
                        userRemoteConfigs: [[
                            url: env.GITHUB_REPO_URL,
                            credentialsId: env.GITHUB_CREDENTIALS_ID
                        ]]
                    ])
                    // Explicitly set GIT_COMMIT if it's not already set
                    if (!env.GIT_COMMIT) {
                        env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                    }
                    echo "Current commit: ${env.GIT_COMMIT}"
                }
            }
        }

        stage('Detect Changes') {
            steps  {
                script {
                    def affectedServices = []
                    // Check for tag build first
                    if (env.TAG_NAME || env.BRANCH_NAME == 'main') { // If a tag is present or push to main, we assume all services are affected
                        if (env.TAG_NAME) {
                            echo "A new release found with tag ${env.TAG_NAME}"
                        } else {
                            echo "Main branch build detected"
                        }
                        affectedServices = VALID_SERVICES
                        AFFECTED_SERVICES = affectedServices.join(' ')
                        echo "Changed services: ${AFFECTED_SERVICES}"
                        return
                    }
                    // Regular build with change detection
                    def changedFiles = sh(
                        script: """
                            # Get the last successful commit hash or use HEAD~1
                            LAST_COMMIT=\$(git rev-parse HEAD~1 2>/dev/null || git rev-parse origin/main)
                            # Get the changed files
                            git diff --name-only \$LAST_COMMIT HEAD
                        """,
                        returnStdout: true
                    ).trim()

                    if (changedFiles == null || changedFiles.trim().isEmpty()) {
                        echo "No changes detected."
                        AFFECTED_SERVICES = ''
                        return
                    }

                    changedFiles.split("\n").each { file ->
                        VALID_SERVICES.each { service ->
                            if (file.startsWith("${service}/") && !affectedServices.contains(service)) {
                                affectedServices.add(service)
                                echo "Detected changes in ${service}"
                            }
                        }
                    }

                    if (affectedServices.isEmpty()) {
                        echo "No valid service changes detected. Skipping pipeline"
                        AFFECTED_SERVICES = ''
                        return
                    }

                    def servicesString = affectedServices.join(' ')
                    AFFECTED_SERVICES = servicesString
                    echo "ENV_AFFECTED_SERVICES: [${AFFECTED_SERVICES}]"
                    echo "Changed services: ${servicesString}"
                }
            }
        }

        stage('Login to DockerHub') {
            when {
                expression { return AFFECTED_SERVICES != '' || env.TAG_NAME != null }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: env.DOCKER_HUB_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                }
            }
        }

        stage('Build and Push Docker Images') {
            when {
                expression { return AFFECTED_SERVICES != '' || env.TAG_NAME != null }
            }
            steps {
                script {
                    def CONTAINER_TAG = ""

                    if (env.TAG_NAME != null) {
                        CONTAINER_TAG = env.TAG_NAME
                    } else if (env.BRANCH_NAME == 'main') {
                        CONTAINER_TAG = 'latest'
                    } else {
                        CONTAINER_TAG = env.GIT_COMMIT.take(7)
                    }

                    echo "Using tag: ${CONTAINER_TAG}"
                    echo "Building images for services: ${AFFECTED_SERVICES}"
                    // Split the string into an array
                    def services = AFFECTED_SERVICES.split(' ')
                    for (service in services) {
                        echo "Building and pushing Docker image for ${service}"
                        sh """
                            cd ${service}
                            mvn clean install -P buildDocker -Dmaven.test.skip=true \\
                                -Ddocker.image.prefix=${env.DOCKER_REGISTRY} \\
                                -Ddocker.image.tag=${CONTAINER_TAG}
                            docker push ${env.DOCKER_REGISTRY}/${service}:${CONTAINER_TAG}
                            cd ..
                        """
                    }
                }
            }
        }

        stage('Deploy k8s') {
            when { expression { return AFFECTED_SERVICES != '' } }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GITHUB_CREDENTIALS_ID, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
                            git clone https://\$GIT_USERNAME:\$GIT_PASSWORD@${env.MANIFEST_REPO} k8s
                            cd k8s
                            git config user.name "Jenkins"
                            git config user.email "jenkins@example.com"
                        """
                    }
                    sh '''
                        cd k8s
                        # Extract old version using grep + cut
                        old_version=$(grep '^version:' Chart.yaml | cut -d' ' -f2)
                        echo "Old version: $old_version"
                        major=$(echo "$old_version" | cut -d. -f1)
                        minor=$(echo "$old_version" | cut -d. -f2)
                        patch=$(echo "$old_version" | cut -d. -f3)
                        new_patch=$((patch + 1))
                        new_version="$major.$minor.$new_patch"
                        echo "New version: $new_version"
                        # Update version using sed
                        sed -i "s/^version: .*/version: $new_version/" Chart.yaml
                    '''

                    def COMMIT_MSG = ""
                    def shouldDeploy = false
                    if (env.TAG_NAME != null) { // check for tag
                        echo "Deploying to staging ${env.TAG_NAME}"
                        COMMIT_MSG = "Deploy for tag: ${env.TAG_NAME}"
                        def services = AFFECTED_SERVICES.split(' ')
                        for (service in services) {
                            def shortName = service.replaceFirst('spring-petclinic-', '')
                            echo "Building and pushing Docker image for ${shortName}"
                            
                            // Get the digest in a single shell command and update the files
                            sh """
                                cd k8s
                                # Get the digest
                                digest=\$(docker inspect --format='{{index .RepoDigests 0}}' ${env.DOCKER_REGISTRY}/${service}:${env.TAG_NAME} | cut -d'@' -f2)
                                echo "Digest for ${shortName}: \$digest"
                                
                                # Update tag
                                sed -i "s/^imageTag: .*/imageTag: \\&tag ${env.TAG_NAME}/" environments/staging-values.yaml
                                
                                # Update digest
                                sed -i "/${shortName}:/,/digest:/ s/digest: .*/digest: \$digest/" environments/staging-values.yaml
                            """
                        }
                        echo "Deploying all services to staging at tag ${env.TAG_NAME}"
                        shouldDeploy = true
                    } else if (env.BRANCH_NAME == 'main') {
                        echo "Deploying to production"
                        AFFECTED_SERVICES.split(' ').each { fullName ->
                            def shortName = fullName.replaceFirst('spring-petclinic-', '')
                            def shortCommit = 'latest'
                            sh """
                                cd k8s
                                sed -i '/${shortName}:/{n;n;s/tag:.*/tag: ${shortCommit}/}' environments/dev-values.yaml
                            """
                            echo "Updated tag for ${shortName} to ${env.GIT_COMMIT.take(7)}"
                        }
                        COMMIT_MSG = "Deploy for branch main with commit latest"
                        shouldDeploy = true
                    } else if (env.BRANCH_NAME.startsWith('develop')) {
                        echo "Deploying to dev"
                        AFFECTED_SERVICES.split(' ').each { fullName ->
                            def shortName = fullName.replaceFirst('spring-petclinic-', '')
                            def shortCommit = env.GIT_COMMIT.take(7)
                            sh """
                                cd k8s
                                sed -i '/${shortName}:/{n;n;s/tag:.*/tag: ${shortCommit}/}' environments/dev-values.yaml
                            """
                            echo "Updated tag for ${shortName} to ${env.GIT_COMMIT.take(7)}"
                        }
                        COMMIT_MSG = "Deploy for branch dev with commit ${env.GIT_COMMIT.take(7)}"
                        shouldDeploy = true
                    } else {
                        echo "Push by developer, manual deploy required"
                        shouldDeploy = false
                    }

                    if (shouldDeploy) {
                        sh """
                            cd k8s
                            git add .
                            git commit -m "${COMMIT_MSG}"
                            git push origin main
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            echo "Workspace cleaned"
            
            sh "docker system prune -af"
            sh "docker logout"
            echo "Docker cleanup and logout completed"
        }
    }
}
