pipeline {
    agent any
    
    tools {
        maven 'maven3'
    }
    
    environment {
        IMAGE_NAME = "adijaiswal/bankapp"
        TAG = "${env.BUILD_NUMBER}" 
        SONAR_SCANNER= tool 'sonar-scanner'
    }

    stages {       
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                echo 'Hello World'
            }
        }
        
        stage('SOnarQube') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SONAR_SCANNER/bin/sonar-scanner -Dsonar.projectKey=Multitier -Dsonar.projectName=Multitier \
                    -Dsonar.java.binaries=target '''
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        
        stage('Trivy FS') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Publish artifacts') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }
        
         stage('Trivy SCan Docker Image') {
            steps {
                sh "trivy image --format table -o image.html ${IMAGE_NAME}:${TAG}"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
        
        stage('Update Kubernetes Manifest') {
            steps {
                script {
                    // Replace the Docker image tag in line 58 of the ds.yml file
                    sh """
                    sed -i '58s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${TAG}|' ds.yml
                    """
                }
            }
        }
        
        stage('Commit and Push Changes') {
    steps {
        script {
            // Use GitHub credentials from Jenkins
            withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                sh """
                git config --global user.email "your-email@example.com"
                git config --global user.name "Aditya Jaiswal"
                git remote set-url origin https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/jaiswaladi246/Multi-Tier-Java.git
                git pull origin main
                git add ds.yml
                git commit -m "Update image to ${IMAGE_NAME}:${TAG}"
                git push origin main
                """
            }
        }
    }
}

         stage('K8 Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://761EA1E7FE48A0FB44798AAB35344BB7.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ds.yml -n webapps"
                        sleep 30
                    }
            }
        }
        
        stage('Verify K8 Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'devopsshack-cluster', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://761EA1E7FE48A0FB44798AAB35344BB7.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl get pods -n webapps"
                        sh "kubectl get svc -n webapps"
                       
                    }
            }
        }
    }
}
