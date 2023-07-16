def jenize_config = 'jenkinize.config'

def environment_parameters = [:]

def project_name = 'Unknown Project'
def project_name_clean =  null
def app_port = 8080

def names = [:]
def paths = [:]

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

                            if (!fileExists(jenize_config)) {
                                error("File ${jenize_config} not found!")
                            }
                            def config = load jenize_config


                            if(config.project_name){
                                project_name = config.project_name
                                echo "Project name has been set in config as ${project_name}"
                            }
                            else {
                                project_name = 'Laravel Project'
                                echo "Project name has not been set in config, defaulting to ${project_name}"
                            }


                            project_name_clean = project_name.toLowerCase().replace(' ', '_')
                            echo "Project \"clean\" name has set to ${project_name_clean}"




                            if(!env.BRANCH_NAME) {
                                error("Environment variable BRANCH_NAME is unavailable.  Contact the developer.")
                            }

                            if (!config.environments || config.environments.isEmpty()) {
                                error("Environments hasn't been setup in config.")
                            }

                            def environment = null
                            config.environments.each { environment_key, parameters ->

                                if (!environment) {
                                    if((parameters.branch_name && env.BRANCH_NAME == parameters.branch_name) ||
                                    (!parameters.branch_name && env.BRANCH_NAME == environment_key)
                                    ) {
                                        names.branch_name = env.BRANCH_NAME
                                        environment_parameters = parameters
                                        environment = environment_key
                                        echo "Environment has been matched to the config: '${environment}' with the parameters: ${environment_parameters}"
                                    }

                                }

                            }

                            if(!environment){
                                error("No valid environment detected for branch '${env.BRANCH_NAME}' in jenkins.config. Failing the pipeline.")
                            }


                            if(environment_parameters.environment_name)
                                names.environment_name = environment_parameters.environment_name
                            else
                                names.environment_name = environment

                            names.environment_name_clean = names.environment_name.toLowerCase().replace(' ', '_')


                            if(environment_parameters.app_port){
                                app_port = environment_parameters.app_port
                                echo "Port found in environment parameters: ${app_port}"
                            }
                            else {
                                echo "No port found in environment parameters, using default: ${app_port}"
                            }




                            names.base_name = "${project_name_clean}_${names.environment_name_clean}"

                            names.app_container_name = "${names.base_name}_app"
                            names.network_name = "${names.base_name}_network"
                            names.codebase_volume_name = "${names.base_name}_codebase"

                            names.composer_project_name = "${names.base_name}"
                            names.secret_file_credentials_id = "${names.base_name}_env"

                            paths.jenkins_deploy_directory = "var/deploy/${names.environment_name_clean}/${names.environment_name_clean}"
                            paths.dockerfile = config.dockerfile
                            paths.docker_compose_file = config.docker_compose_file

                            paths.app_codebase_directory = "/var/www/html/"
                            paths.app_env_file = "${paths.app_codebase_directory}.env"


                            echo "Name variables have been set: ${names}"
                            echo "Path variables have been set: ${paths}"


                        }
                    }
                }

                stage('Confirm Deployment') {
                    when {
                        expression {
                            environment_parameters.require_confirmation
                        }
                    }
                    steps {
                        script {

                            input(message: "Are you sure you want to deploy to ${names.environment_name}?")
                            //asdf
                        }
                    }
                }
                stage('Sync Code') {
                    steps {
                        script {
                            echo "Contents of the source directory:"
                            sh "ls -la"
                            sh "mkdir -p ${paths.jenkins_deploy_directory}"
                            sh "rsync -av --exclude='.git' --exclude='node_modules' --exclude='vendor' --delete . ${paths.jenkins_deploy_directory}"
                            echo "Contents of the Jenkins deploy directory:"
                            sh "ls -la ${paths.jenkins_deploy_directory}"
                        }
                    }
                }
                stage('Build Docker Image') {
                    steps {
                        script {
                            def imageExistsOutput = sh(script: "docker images -q ${names.app_container_name}:latest | wc -l", returnStdout: true).trim().toInteger()
                            def dockerfileChangedOutput = sh(script: "git diff HEAD^ HEAD --name-only | grep Dockerfile | wc -l", returnStdout: true).trim().toInteger()
                            def imageExists = imageExistsOutput > 0
                            def dockerfileChanged = dockerfileChangedOutput > 0
                            if (!imageExists || dockerfileChanged) {
                                echo "Building Docker image: ${names.app_container_name}"
                                sh "docker build -t ${names.app_container_name} -f ${paths.dockerfile} ."
                            } else {
                                echo "Docker image ${names.app_container_name} is up to date. Skipping build."
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
                            echo "Stopping old container: ${names.app_container_name}"
                            sh "docker stop ${names.app_container_name} || true"
                            sh "docker rm ${names.app_container_name} || true"
                        }
                    }
                }
                stage('Delete and recreate the codebase volume.') {
                    steps {
                        script {
                            echo "Recreating codebase volume for: ${names.app_container_name}"
                            sh "docker volume rm ${names.codebase_volume_name} || true"
                            sh "docker volume create --name ${names.codebase_volume_name}"
                        }
                    }
                }
                stage('Create Network') {
                    steps {
                        script {
                            echo "Creating network: ${names.network_name}"
                            sh "docker network create ${names.network_name} || true"
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
                            echo "Starting new container: ${names.app_container_name}"
                            sh "export CONTAINER_NAME=${names.app_container_name} && export APP_PORT=${app_port} && docker-compose -p ${names.composer_project_name} -f ${paths.docker_compose_file} up -d"
                        }
                    }
                }
                stage('Copy Code to Deployment Environment') {
                    steps {
                        script {
                            sh "docker cp ${paths.jenkins_deploy_directory}/. ${names.app_container_name}:/var/www/html/"
                        }
                    }
                }
                stage('Rename Environment File') {
                    steps {
                        script {
                            def envFilePath = "${paths.app_codebase_directory}.env.${names.environment_name_clean}"
                            echo "Renaming ${envFilePath} to ${paths.app_env_file}"

                            def fileExistsOutput = sh(script: "docker exec ${names.app_container_name} test -f ${envFilePath} && echo 'true' || echo 'false'", returnStdout: true).trim()

                            if (fileExistsOutput == 'true') {
                                sh "docker exec ${names.app_container_name} mv ${envFilePath} ${paths.app_env_file}"
                                echo "File renamed successfully."
                            } else {
                                error("The .env file for this environment does not exist.  It should be named '.env.${names.environment_name_clean}'")
                            }
                        }
                    }
                }
                stage('Update Environment Variables') {
                    steps {
                        script {
                            // Store the original permissions and ownership
                            def originalPermissions = sh(script: "docker exec ${names.app_container_name} stat -c '%a' ${paths.app_env_file}", returnStdout: true).trim()
                            def originalOwnership = sh(script: "docker exec ${names.app_container_name} ls -l ${paths.app_env_file} | awk '{print \$3\":\"\$4}'", returnStdout: true).trim()
                            // Change the ownership and permissions of the .env file
                            sh "docker exec ${names.app_container_name} chown www-data:www-data ${paths.app_env_file}"
                            sh "docker exec ${names.app_container_name} chmod 664 ${paths.app_env_file}"
                            // Append the secret file to .env
                            catchError {
                                withCredentials([file(credentialsId: "${names.secret_file_credentials_id}", variable: 'SECRET_FILE')]) {
                                    sh "docker exec -i ${names.app_container_name} bash -c 'cat >> ${paths.app_env_file}' < ${SECRET_FILE}"
                                }
                            }
                            // Restore the original permissions and ownership
                            sh "docker exec ${names.app_container_name} chown ${originalOwnership} ${paths.app_env_file}"
                            sh "docker exec ${names.app_container_name} chmod ${originalPermissions} ${paths.app_env_file}"
                        }
                    }
                }
                stage('Fix Storage Permissions') {
                    steps {
                        script {
                            sh "docker exec ${names.app_container_name} chown -R www-data:www-data ${paths.app_codebase_directory}storage"
                            sh "docker exec ${names.app_container_name} chown -R www-data:www-data ${paths.app_codebase_directory}bootstrap/cache"
                            sh "docker exec ${names.app_container_name} chmod -R 775 ${paths.app_codebase_directory}storage"
                            sh "docker exec ${names.app_container_name} chmod -R 775 ${paths.app_codebase_directory}bootstrap/cache"
                        }
                    }
                }
                stage('Composer Install') {
                    steps {
                        script {
                            sh "docker exec ${names.app_container_name} composer install"
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
                            sh "docker exec -w /var/www/html ${names.app_container_name} php artisan migrate --force"
                        }
                    }
                }
                stage('Other Tasks') {
                    steps {
                        script {
                            echo "Running Other Tasks"
                            sh "docker exec -w ${paths.app_codebase_directory} ${names.app_container_name} php artisan config:cache"
                            sh "docker exec -w ${paths.app_codebase_directory} ${names.app_container_name} php artisan route:cache"
                            sh "docker exec -w ${paths.app_codebase_directory} ${names.app_container_name} php artisan view:cache"
                        }
                    }
                }
            }
        }
    }
}
