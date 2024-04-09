pipeline {
    agent any
    //tools{
     //   jdk 'jdk17'
       // nodejs 'node18'
    //  }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Netflix-clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
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
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build") {
             steps {
                 sh 'docker build . --build-arg TMDB_V3_API_KEY=556921937e1be1e4703fbe797151c3e0 -t netflix'
             }
        }
        stage("Docker Push"){
            steps{
                script{
                   //withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                     //  sh "sudo docker build --build-arg TMDB_V3_API_KEY=556921937e1be1e4703fbe797151c3e0 -t netflix ."
                      // sh "sudo docker tag netflix dockers766/netflix:latest "
                       //sh "sudo docker push dockers766/netflix:latest "
                       
                    withCredentials([usernamePassword(credentialsId: "docker", passwordVariable: "dockerhubPass", usernameVariable: "dockerhubUser")]) {
                     sh "docker tag netflix ${env.dockerhubUser}/netflix:latest"
                     sh "docker login -u ${env.dockerhubUser} -p ${env.dockerhubPass}"
                     sh "docker push ${env.dockerhubUser}/netflix:latest"
                       
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image dockers766/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 dockers766/netflix:latest'
            }
        }

        stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }
                
        
    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'kodithyala.ashok@gmail.com',
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
