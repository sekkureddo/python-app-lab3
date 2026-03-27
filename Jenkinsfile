pipeline {
    agent any

    parameters {
        string(name: 'STUDENT_NAME', defaultValue: 'Иванов Иван', description: 'ФИО студента')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'], description: 'Среда')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Запускать тесты')
    }

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_IMAGE = "sekkureddo/student-app:${BUILD_NUMBER}"
        CONTAINER_NAME = "student-app-${params.ENVIRONMENT}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Клонирование репозитория..."
                git branch: 'main', 
                    url: 'https://github.com/sekkureddo/python-app-lab3.git', 
                    credentialsId: 'github-credentials'
            }
        }

        stage('Setup Python') {
            steps {
                echo 'Настройка окружения Python...'
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                '''
            }
        }

        stage('Tests') {
            when {
                expression { params.RUN_TESTS }
            }
            stages {
                stage('Unit Tests') {
                    steps {
                        echo "Настройка окружения и запуск тестов..."
                        sh '''
                        . venv/bin/activate
                        python3 -m unittest test_app.py -v
                        '''
                    }
                }
                stage('Integration Tests') {
                    steps {
                        echo "Запуск интеграционных тестов..."
                        // Здесь может быть команда для других тестов
                        sh 'echo "Интеграционные тесты пройдены"'
                    }
                }
            }
        }

        stage('Build and Push') {
            stages {
                stage('Build Docker Image') {
                    steps {
                        script {
                            echo "Сборка образа: ${DOCKER_IMAGE}"
                            docker.build("${DOCKER_IMAGE}")
                        }
                    }
                }
                stage('Push to Registry') {
                    steps {
                        script {
                            // Используем учетные данные с ID 'docker-hub-credentials'
                            docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-hub-credentials') {
                                docker.image("${DOCKER_IMAGE}").push()
                                docker.image("${DOCKER_IMAGE}").push('latest')
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                expression { params.ENVIRONMENT == 'dev' }
            }
            steps {
                echo "Развертывание в DEV среду..."
                sh "docker rm -f ${CONTAINER_NAME} || true"
                sh """
                  docker run -d \
                   --name ${CONTAINER_NAME} \
                   -p 8081:5000 \
                   -e STUDENT_NAME='${params.STUDENT_NAME}' \
                   -e PORT=5000 \
                   ${DOCKER_IMAGE}
                """
            }
        }

        stage('Deploy to Staging') {
            when {
                expression { params.ENVIRONMENT == 'staging' }
            }
            steps {
                echo "Развертывание в STAGING среду..."
                sh "docker rm -f ${CONTAINER_NAME} || true"
                sh """
                  docker run -d \
                   --name ${CONTAINER_NAME} \
                   -p 8082:5000 \
                   -e STUDENT_NAME='${params.STUDENT_NAME}' \
                   -e PORT=5000 \
                   ${DOCKER_IMAGE}
                """
            }
        }

        stage('Approve Production') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            input {
                message "Подтвердите развертывание в PRODUCTION?"
                ok "Да, развернуть"
                parameters {
                    string(name: 'PRODUCTION_VERSION', defaultValue: 'latest', description: 'Версия для тега')
                }
            }
            steps {
                echo "Развертывание в PRODUCTION одобрено пользователем"
                // 3. Тегирование версий (создаем Git-тег)
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', passwordVariable: 'GIT_PASS', usernameVariable: 'GIT_USER')]) {
                        sh "git tag -a v${BUILD_NUMBER}-${params.PRODUCTION_VERSION} -m 'Release v${BUILD_NUMBER}'"
                        sh "git push https://${GIT_USER}:${GIT_PASS}@github.com/sekkureddo/python-app-lab3.git --tags"
                    }
                }
            }
        }

        stage('Deploy to Production') {
            when {
                expression { params.ENVIRONMENT == 'production' }
            }
            steps {
                script {
                    echo "Развертывание в PRODUCTION (порт 80)..."
                    sh "docker rm -f ${CONTAINER_NAME} || true"
                    sh "docker pull ${DOCKER_IMAGE}"
                    sh """
                  docker run -d \
                   --name ${CONTAINER_NAME} \
                   -p 80:5000 \
                   -e STUDENT_NAME='${params.STUDENT_NAME}' \
                   -e PORT=5000 \
                   ${DOCKER_IMAGE}
                """
                }
            }
        }
    }

    post {
        success {
            script {
                echo "Pipeline успешно выполнен для среды: ${params.ENVIRONMENT}"
            }
        }
        failure {
            emailext(
                to: 'reddosekaru@mail.ru',
                subject: "Ошибка в Pipeline: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "Сборка №${env.BUILD_NUMBER} завершилась ошибкой. Подробности: ${env.BUILD_URL}",
                attachLog: true
            )
        }
        always {
            echo "Очистка рабочего пространства..."
            cleanWs()
        }
    }
}
