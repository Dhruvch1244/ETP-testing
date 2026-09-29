// ==============================================================================
// ETP-testing CI pipeline
//   1. SCM checkout (public GitHub repo)
//   2. Docker build for every service image
//   3. Docker push to JFrog Artifactory
//   4. docker compose up  — OPTIONAL, gated behind a parameter + manual
//      approval, since it stands up long-lived containers on the agent
//
// One-time manual setup this pipeline assumes already exists in Jenkins —
// see the "Manual setup" notes returned alongside this file. Nothing here
// hardcodes secrets; everything sensitive comes from Jenkins credentials.
// ==============================================================================

pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        string(
            name: 'REPO_URL',
            defaultValue: 'https://github.com/IM-RAHUL-RAJ/ETP-testing',
            description: 'Public repo to clone'
        )
        string(
            name: 'REPO_BRANCH',
            defaultValue: 'rimjhim_testing',
            description: 'Branch to build'
        )
        string(
            name: 'ARTIFACTORY_REGISTRY',
            defaultValue: '',
            description: 'JFrog Artifactory Docker registry host, e.g. mycompany.jfrog.io'
        )
        string(
            name: 'ARTIFACTORY_REPO',
            defaultValue: 'docker-local',
            description: 'Artifactory Docker repository name'
        )
        booleanParam(
            name: 'PUSH_IMAGES',
            defaultValue: true,
            description: 'Push built images to Artifactory'
        )
        booleanParam(
            name: 'RUN_DOCKER_COMPOSE',
            defaultValue: false,
            description: 'Permission gate: after build/push, bring the full stack up on this agent via docker compose. Requires manual approval when true.'
        )
    }

    environment {
        JFROG_CREDS_ID   = 'jfrog-creds'
        ENV_FILE_CREDS_ID = 'etp-dotenv'
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('SCM Checkout') {
            steps {
                echo "Cloning ${params.REPO_URL} @ ${params.REPO_BRANCH}"
                git branch: params.REPO_BRANCH, url: params.REPO_URL
            }
        }

        stage('Tool Sanity Check') {
            steps {
                sh '''
                    docker version
                    docker compose version
                '''
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    def services = [
                        [name: 'auth-service', context: 'sprint8-auth-service', dockerfile: 'Dockerfile'],
                        [name: 'trade-api',    context: 'sprint8',              dockerfile: 'Dockerfile'],
                        [name: 'executor',     context: 'executor',             dockerfile: 'Dockerfile'],
                        [name: 'frontend',     context: 'sprint8/front-end',    dockerfile: 'Dockerfile'],
                    ]
                    services.each { svc ->
                        echo "Building ${svc.name} from ${svc.context}"
                        sh """
                            docker build -f ${svc.context}/${svc.dockerfile} \
                                -t etp-${svc.name}:${IMAGE_TAG} \
                                -t etp-${svc.name}:latest \
                                ${svc.context}
                        """
                    }
                }
            }
        }

        stage('Push to JFrog Artifactory') {
            when {
                expression { params.PUSH_IMAGES }
            }
            steps {
                script {
                    if (!params.ARTIFACTORY_REGISTRY?.trim()) {
                        error("ARTIFACTORY_REGISTRY parameter must be set to push images.")
                    }
                    def services = ['auth-service', 'trade-api', 'executor', 'frontend']
                    withCredentials([usernamePassword(
                        credentialsId: env.JFROG_CREDS_ID,
                        usernameVariable: 'JFROG_USER',
                        passwordVariable: 'JFROG_PASS'
                    )]) {
                        sh 'echo "$JFROG_PASS" | docker login "$ARTIFACTORY_REGISTRY" -u "$JFROG_USER" --password-stdin'
                    }
                    services.each { svc ->
                        def target = "${params.ARTIFACTORY_REGISTRY}/${params.ARTIFACTORY_REPO}/etp-${svc}"
                        sh """
                            docker tag etp-${svc}:${IMAGE_TAG} ${target}:${IMAGE_TAG}
                            docker tag etp-${svc}:${IMAGE_TAG} ${target}:latest
                            docker push ${target}:${IMAGE_TAG}
                            docker push ${target}:latest
                        """
                    }
                    sh 'docker logout "$ARTIFACTORY_REGISTRY"'
                }
            }
        }

        stage('Approve Docker Compose Up') {
            when {
                expression { params.RUN_DOCKER_COMPOSE }
            }
            steps {
                input message: "Bring the full stack up on agent '${env.NODE_NAME}' via docker compose?",
                      ok: 'Proceed'
            }
        }

        stage('Docker Compose Up') {
            when {
                expression { params.RUN_DOCKER_COMPOSE }
            }
            steps {
                script {
                    try {
                        withCredentials([file(credentialsId: env.ENV_FILE_CREDS_ID, variable: 'DOTENV_FILE')]) {
                            sh 'cp "$DOTENV_FILE" .env'
                        }
                        echo "Using .env from Jenkins credential '${env.ENV_FILE_CREDS_ID}'."
                    } catch (err) {
                        echo "No '${env.ENV_FILE_CREDS_ID}' credential configured — falling back to the defaults baked into docker-compose.yml."
                    }
                    sh 'docker compose up -d --build'
                    sh 'docker compose ps'
                }
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
        }
        failure {
            echo 'Pipeline failed — see stage logs above.'
        }
    }
}
