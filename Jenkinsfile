pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
    }
    environment {
        PROJECT_NAME = 'UnknownProject'
        BASE_NAME = ''
        JENKINS_DEPLOY_DIRECTORY = ''
        CODEBASE_VOLUME_NAME = ''
        CURRENT_ENV = ''
        CONTAINER_NAME = ''
        NETWORK_NAME = ''
        COMPOSER_PROJECT_NAME = ''
        SECRET_FILE_CREDENTIALS_ID = ''
        APP_PORT = ''
    }
    stages {
        stage('Prepare') {
            stages {
                stage('Checkout') {
                    steps {
                        cleanWs()
                        checkout scm
                    }
                }
                stage('Load Config') {
                    steps {
                        def config = load 'jenkins.config'
                    }
                }
                stage('Determine Environment') {
                    steps {
                        script {
                            switch (env.BRANCH_NAME) {
                                case 'test':
                                CURRENT_ENV = 'test'
                                echo "Detected branch: ${env.BRANCH_NAME}. Setting environment to 'test'."
                                break case 'stage':
                                CURRENT_ENV = 'stage'
                                echo "Detected branch: ${env.BRANCH_NAME}. Setting environment to 'stage'."
                                break case 'main':
                                CURRENT_ENV = 'prod'
                                echo "Detected branch: ${env.BRANCH_NAME}. Setting environment to 'prod'."
                                break default:
                                CURRENT_ENV = null
                                echo "Detected branch: ${env.BRANCH_NAME}. Skipping remaining steps."
                            }
                            if (CURRENT_ENV == null) {
                                error("No valid environment detected for branch ${env.BRANCH_NAME}. Failing the pipeline.")
                            }
                        }
                    }
                }
                stage('Loading Environment Configuration') {
                    steps {
                        script {

                                BASE_NAME = "${PROJECT_NAME}_${CURRENT_ENV}"
                                CONTAINER_NAME = "${BASE_NAME}_app"
                                NETWORK_NAME = "${BASE_NAME}_network"
                                CODEBASE_VOLUME_NAME = "${BASE_NAME}_codebase"
                                JENKINS_DEPLOY_DIRECTORY = "var/deploy/${CURRENT_ENV}/${CURRENT_ENV}"
                                COMPOSER_PROJECT_NAME = "${BASE_NAME}"
                                SECRET_FILE_CREDENTIALS_ID = "${BASE_NAME}_env"
                                switch (CURRENT_ENV) {
                                    case 'test':
                                        APP_PORT = '8002'
                                    break case 'stage':
                                        APP_PORT = '8001'
                                    break case 'prod':
                                        APP_PORT = '8000'
                                }                            }
                        }
                    }
                }
                stage('Confirm Production Deployment') {
                    when {
                        expression {
                            CURRENT_ENV == 'prod'
                        }
                    }
                    steps {
                        script {
                            echo 'Skipping for debugging...'
                            //input(message: 'Are you sure you want to deploy to production?')
                            //asdf
                        }
                    }
                }
                stage('Sync Code') {
                    steps {
                        script {
                            echo "Contents of the source directory:"
                            sh "ls -la"
                            sh "mkdir -p ${JENKINS_DEPLOY_DIRECTORY}"
                            sh "rsync -av --exclude='.git' --exclude='node_modules' --exclude='vendor' --delete . ${JENKINS_DEPLOY_DIRECTORY}"
                            echo "Contents of the Jenkins deploy directory:"
                            sh "ls -la ${JENKINS_DEPLOY_DIRECTORY}"
                        }
                    }
                }
                stage('Build Docker Image') {
                    steps {
                        script {
                            def imageExistsOutput = sh(script: "docker images -q ${CONTAINER_NAME}:latest | wc -l", returnStdout: true).trim().toInteger()
                            def dockerfileChangedOutput = sh(script: "git diff HEAD^ HEAD --name-only | grep Dockerfile | wc -l", returnStdout: true).trim().toInteger()
                            def imageExists = imageExistsOutput > 0
                            def dockerfileChanged = dockerfileChangedOutput > 0
                            if (!imageExists || dockerfileChanged) {
                                echo "Building Docker image: ${CONTAINER_NAME}"
                                sh "docker build -t ${CONTAINER_NAME} ."
                            } else {
                                echo "Docker image ${CONTAINER_NAME} is up to date. Skipping build."
                            }
                        }
                    }
                }
            }
        }
        stage('Setup') {
            stages {
                stage('Stop Old Container') {
                    steps {
                        script {
                            echo "Stopping old container: ${CONTAINER_NAME}"
                            sh "docker stop ${CONTAINER_NAME} || true"
                            sh "docker rm ${CONTAINER_NAME} || true"
                        }
                    }
                }
                stage('Delete and recreate the codebase volume.') {
                    steps {
                        script {
                            echo "Recreating codebase volume for: ${CONTAINER_NAME}"
                            sh "docker volume rm ${CODEBASE_VOLUME_NAME} || true"
                            sh "docker volume create --name ${CODEBASE_VOLUME_NAME}"
                        }
                    }
                }
                stage('Create Network') {
                    steps {
                        script {
                            echo "Creating network: ${NETWORK_NAME}"
                            sh "docker network create ${NETWORK_NAME} || true"
                        }
                    }
                }
            }
        }
        stage('Deploy') {
            stages {
                stage('Start New Container') {
                    steps {
                        script {
                            echo "Starting new container: ${CONTAINER_NAME}"
                            sh "export CONTAINER_NAME=${CONTAINER_NAME} && export APP_PORT=${APP_PORT} && docker-compose -p ${COMPOSER_PROJECT_NAME} -f docker-compose.yml up -d"
                        }
                    }
                }
                stage('Copy Code to Deployment Environment') {
                    steps {
                        script {
                            sh "docker cp ${JENKINS_DEPLOY_DIRECTORY}/. ${CONTAINER_NAME}:/var/www/html/"
                        }
                    }
                }
                stage('Rename Environment File') {
                    steps {
                        script {
                            def envFilePath = "/var/www/html/.env.${CURRENT_ENV}"
                            echo "Renaming ${envFilePath} to /var/www/html/.env"
                            sh "docker exec ${CONTAINER_NAME} mv ${envFilePath} /var/www/html/.env"
                        }
                    }
                }
                stage('Update Environment Variables') {
                    steps {
                        script {
                            // Store the original permissions and ownership
                            def originalPermissions = sh(script: "docker exec ${CONTAINER_NAME} stat -c '%a' /var/www/html/.env", returnStdout: true).trim()
                            def originalOwnership = sh(script: "docker exec ${CONTAINER_NAME} ls -l /var/www/html/.env | awk '{print \$3\":\"\$4}'", returnStdout: true).trim()
                            // Change the ownership and permissions of the .env file
                            sh "docker exec ${CONTAINER_NAME} chown www-data:www-data /var/www/html/.env"
                            sh "docker exec ${CONTAINER_NAME} chmod 664 /var/www/html/.env"
                            // Append the secret file to .env
                            withCredentials([file(credentialsId: "${SECRET_FILE_CREDENTIALS_ID}", variable: 'SECRET_FILE')]) {
                                sh "docker exec -i ${CONTAINER_NAME} bash -c 'cat >> /var/www/html/.env' < ${SECRET_FILE}"
                            }
                            // Restore the original permissions and ownership
                            sh "docker exec ${CONTAINER_NAME} chown ${originalOwnership} /var/www/html/.env"
                            sh "docker exec ${CONTAINER_NAME} chmod ${originalPermissions} /var/www/html/.env"
                        }
                    }
                }
                stage('Fix Storage Permissions') {
                    steps {
                        script {
                            sh "docker exec ${CONTAINER_NAME} chown -R www-data:www-data /var/www/html/storage"
                            sh "docker exec ${CONTAINER_NAME} chown -R www-data:www-data /var/www/html/bootstrap/cache"
                            sh "docker exec ${CONTAINER_NAME} chmod -R 775 /var/www/html/storage"
                            sh "docker exec ${CONTAINER_NAME} chmod -R 775 /var/www/html/bootstrap/cache"
                        }
                    }
                }
                stage('Composer Install') {
                    steps {
                        script {
                            sh "docker exec ${CONTAINER_NAME} composer install"
                        }
                    }
                }
            }
        }
        stage('Finalize') {
            stages {
                stage('Migrations') {
                    steps {
                        script {
                            echo "Migrating"
                            sh "docker exec -w /var/www/html ${CONTAINER_NAME} php artisan migrate --force"
                        }
                    }
                }
                stage('Other Tasks') {
                    steps {
                        script {
                            echo "Running Other Tasks"
                            sh "docker exec -w /var/www/html ${CONTAINER_NAME} php artisan config:cache"
                            sh "docker exec -w /var/www/html ${CONTAINER_NAME} php artisan route:cache"
                            sh "docker exec -w /var/www/html ${CONTAINER_NAME} php artisan view:cache"
                        }
                    }
                }
            }
        }
    }
}
