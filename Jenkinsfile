def BRANCH_SLUG = BRANCH_NAME.toLowerCase().replaceAll("[^A-Za-z0-9]","-")
env.BRANCH_SLUG = BRANCH_SLUG

node {

    def appImage
    env.DOCKER_REGISTRY = "nexus.kanaridi.com:9000"
    env.PROJECT_NAME = "release-demo-2"
    env.APP_IMAGE_NAME = "${DOCKER_REGISTRY}/repository/kanaridocker/${PROJECT_NAME}:${BRANCH_SLUG}"
    try {
        stage('Checkout') {
            if (env.BRANCH_NAME ==~ /(PR-.*|development|master)/ || env.TAG_NAME ==~ /(.*)/) {
                checkout scm
                //scmSkip(deleteBuild: false, skipPattern:'.*\\[skip ci\\].*')
                //sleep 5
            }
        }
        stage('Build') {
            if (env.BRANCH_NAME ==~ /(PR-.*|development|master)/ || env.TAG_NAME ==~ /(.*)/) {
                withCredentials([
                    // .npmrc file with access to https://nexus.kanaridi.com/repository/npm-group/ registry
                    // nexus registry contains npm proxy and Kanari private registries
                    file(credentialsId: 'nexus_npm_group_config', variable: 'NEXUS_NPM_CONFIG')
                    ]) {
                        sh 'cp \$NEXUS_NPM_CONFIG .npmrc'
                        docker.withRegistry("https://${env.DOCKER_REGISTRY}", 'nexus_docker') {
                            appImage = docker.build("${env.APP_IMAGE_NAME}")
                        }
                    }
            }
        }
        stage('Push image') {
            /* limiting to to tags only */
            if (env.TAG_NAME ==~ /(.*)/) {
                appImage.push()
            }
        }
        stage('Deploy Dev') {
            /* limiting to development tags only */
            if (env.TAG_NAME ==~ /(.*development.*)/) {
                withEnv(['DEPLOY_IMAGE=repository/kanaridocker/deploy-image:latest']) {
                    withCredentials([
                        // 'aks_config' creds used in deployment - this is k8s config
                        file(credentialsId: 'aks_config', variable: 'AKS_CONFIG')
                        ]) {
                        docker.withRegistry("https://${DOCKER_REGISTRY}", 'nexus_docker') {
                            docker.image("${DEPLOY_IMAGE}").pull()
                            docker.image("${DEPLOY_IMAGE}").inside {
                                sh '''
                                    helm upgrade \
                                        ${PROJECT_NAME} \
                                        deployment \
                                        --kubeconfig ${AKS_CONFIG} \
                                        --namespace dev \
                                        --install \
                                        --wait \
                                        --timeout 5m0s \
                                        --atomic \
                                        --cleanup-on-fail \
                                        --set image=${APP_IMAGE_NAME} \
                                        --set global.env=dev \
                                        --set pullPolicy=Always \
                                        --set pullSecret=nexus
                                '''
                            }
                        }    
                    }
                }
            }
        }
        stage('GitHub Release') {
            /* limiting to development and mastr branches */
            if (env.BRANCH_NAME ==~ /(development|master)/) {
                withEnv(['RELEASE_IMAGE=repository/kanaridocker/semantic-release:latest']) {
                    withCredentials([
                        // IMPORTANT NOTE: Now GitHub Token is personal one and linked to aokhotnikov account - please change it to service one
                        string(credentialsId: 'gh_token', variable: 'GH_TOKEN'),
                        // .npmrc file with access to https://nexus.kanaridi.com/repository/npm-group/ registry
                        // nexus registry contains npm proxy and Kanari private registries
                        file(credentialsId: 'nexus_npm_group_config', variable: 'NEXUS_NPM_CONFIG')
                        ]) {
                        docker.withRegistry("https://${DOCKER_REGISTRY}", 'nexus_docker') {
                            docker.image("${RELEASE_IMAGE}").pull()
                            docker.image("${RELEASE_IMAGE}").inside {
                                sh '''
                                    cp \$NEXUS_NPM_CONFIG .npmrc
                                    export GH_TOKEN=${GH_TOKEN}
                                    export CI=true
                                    semantic-release
                                '''
                            }
                        }
                    }
                }
            }
        }
    } finally {
        sh '''
            docker rmi "${APP_IMAGE_NAME}" || true
        '''
    }
}
