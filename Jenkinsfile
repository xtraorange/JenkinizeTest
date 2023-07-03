def environment = null
def environment_parameters = []

def project_name = 'Unknown Project'
def project_name_clean =  null

def names = new ArrayList()
def paths = new ArrayList()

pipeline {
    agent any
    options {
        skipDefaultCheckout(true)
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
                stage('Loading Environment and Configuration') {
                    steps {
                        script {

                            if (!fileExists('jenkins.config')) {
                                error("File jenkins.config not found!")
                            }
                            def config = load 'jenkins.config'


                            if(config.project_name)
                                project_name = config.project_name
                            else
                                project_name = 'Laravel Project'

                            project_name_clean = project_name.toLowerCase().replace(' ', '_')

                            echo "Project name has been set to ${project_name}"



                            if(!env.BRANCH_NAME) {
                                error("Enviromnment variable BRANCH_NAME is unavailable.  Contact the developer.")
                            }

                            if (!config.environments || config.environments.isEmpty()) {
                                error("Environments hasn't been setup in config.")
                            }

                            environment = null
                            config.environments.each { environment_key, data ->

                                if (!environment) {
                                    if((data.branch_name && env.BRANCH_NAME == data.branch_name) ||
                                    (!data.branch_name && env.BRANCH_NAME == environment_key)
                                    ) {
                                        environment = environment_key
                                        environment_parameters = data
                                    }
                                    
                                }
                            
                            }

                            if(!environment){
                                error("No valid environment detected for branch ${env.BRANCH_NAME} in jenkins.config. Failing the pipeline.")
                            }



                            names.base_name = "${project_name_clean}_${environment}"
                            names.container_name = "${names.base_name}_app"
                            names.network_name = "${names.base_name}_network"
                            
                            names.codebase_volume_name = "${names.base_name}_codebase"

                            names.composer_project_name = "${names.base_name}"
                            names.secret_file_credentials_id = "${names.base_name}_env"
                            
                            paths.jenkins_deploy_directory = "var/deploy/${environment}/${environment}"



                        }
                    }
                }

                stage('Confirm Production Deployment') {
                    when {
                        expression {
                            environment_parameters.require_confirmation
                        }
                    }
                    steps {
                        script {
                            echo 'Normally, this would be where you would confirm the deployment, but it\'s currently turned off.'
                            //input(message: 'Are you sure you want to deploy to production?')
                            //asdf
                        }
                    }
                }
                stage('Sync Code') {
                    steps {
                        script {
                            error('stopping the pipeline here')
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
