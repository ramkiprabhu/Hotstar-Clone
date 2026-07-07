pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/ramkiprabhu/Hotstar-Clone.git'
            }
        }
        stage("Sonarqube Analysis ") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Hotstar \
                    -Dsonar.projectKey=Hotstar'''
                }
            }
        }
        stage("quality gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                // Fixed: Added --failOnError false and a backup datafeed path to bypass the NVD rate-limiting crash
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --nvdDatafeed https://jeremylong.github.io/DependencyCheck/gzipped/nvdcve-1.1.json.gz --failOnError false', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Docker Scout FS') {
            steps {
                script {
                    // Fixed: Replaced withDockerRegistry with native host docker login
                    withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh "echo \$PASS | docker login -u \$USER --password-stdin"
                        sh 'docker-scout quickview fs://.'
                        sh 'docker-scout cves fs://.'
                    }
                }   
            }
        }
        stage("Docker Build & Push") {
            steps {
                script {
                    // Fixed: Replaced withDockerRegistry with native host docker execution
                    withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh "echo \$PASS | docker login -u \$USER --password-stdin"
                        sh "docker build -t hotstar ."
                        sh "docker tag hotstar ramkiprabhu28/hotstar:latest"
                        sh "docker push ramkiprabhu28/hotstar:latest"
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script {
                    // Fixed: Replaced withDockerRegistry with native host execution
                    withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        sh "echo \$PASS | docker login -u \$USER --password-stdin"
                        sh 'docker-scout quickview ramkiprabhu28/hotstar:latest'
                        sh 'docker-scout cves ramkiprabhu28/hotstar:latest'
                        sh 'docker-scout recommendations ramkiprabhu28/hotstar:latest'
                    }
                }   
            }
        }
        stage("deploy_docker") {
            steps {
                // Added container cleanup to prevent "Container name already in use" errors on successive runs
                sh "docker rm -f hotstar || true"
                sh "docker run -d --name hotstar -p 3000:3000 ramkiprabhu28/hotstar:latest"
            }
        }
    }
}
