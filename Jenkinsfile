pipeline{
    agent any
    
    options{
        timestamps()
    }
    
    environment{
        SONAR_SCANNER_HOME = tool 'sonarqube-scanner'
        PROJECT_NAME = 'devsecopsdemo'
        USERNAME = "marutijogdand7"
        REPONAME = "devsecops-demo"
        IMAGENAME = "${env.USERNAME}/${env.REPONAME}:v${env.BUILD_NUMBER}"
    }
    
    parameters{
        choice(name: 'ACTION', choices: ['Create', 'Destroy'], description: 'Specify the action')
        string(name: 'SCMRepoURL', defaultValue: 'https://github.com/marutijogdand17/DevSecOpsDemoApp.git', description: 'SCM Repo URL' )
        choice(name: 'ZAPScanType', choices: ['baseline', 'full', 'api'], description: 'Specify OWASP Zap Scan type')
    }
    
    stages {
        
        stage('Git Checkout'){
            
            when{
                expression{
                    params.ACTION == 'Create'
                }
            }
            
            steps{
                git credentialsId: 'git-token', url: params.SCMRepoURL
            }
        }
        
        
        stage('Static Code Analysis: SonarQube'){
            
            when{
                expression{
                    params.ACTION == 'Create'
                }
            }
            
            steps{
                
                // sonarqube-server is the name configured in managed systems
               withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        $SONAR_SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.projectKey=$PROJECT_NAME \
                        '''
               } 
            }
        }
        
        
        stage("Quality Gate: SonarQube"){
            
            when{
                expression{
                    params.ACTION == 'Create'
                }
            }
            
            options{
                timeout(time: 5, unit: 'MINUTES')
            }
            
            steps {
                waitForQualityGate abortPipeline: true, credentialsId: 'sonarqube-token'
            }
        }
        
        stage("Owasp Dependency checker"){
            when{
                expression{
                    params.ACTION == "Create"
                }
            }
            
            steps {
                
                dependencyCheck additionalArguments: ''' 
                    -o './'
                    -s './'
                    -f 'ALL' 
                    --prettyPrint''', odcInstallation: 'OWASP Dependency Checker'
        
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }
        
        stage("Build Docker file"){
            
            when{
                expression{
                    params.ACTION == 'Create'
                }
            }
            
            
            steps{
                sh "docker build -t ${env.IMAGENAME} ."
            }
        }
        
        stage('DAST: OWASP Zap'){
            
            when{
                expression{
                    params.ACTION == 'Create'
                }
            }
            
            
            steps{
               sh 'docker run --add-host=mongoservice:172.17.0.1 -itd --name keynote -p 5555:8080 -e MONGO_URL=mongodb://mongoservice:27017/dev $IMAGENAME'
               
               sh 'docker run -t owasp/zap2docker-stable zap-baseline.py -t http://$(ip -f inet -o addr show docker0 | awk \'{print $4}\' | cut -d \'/\' -f 1):5555/ -m 3 -i'
            }
        }
        
        stage('Push Docker image: DockerHub'){
            
            when{
                expression{
                    params.ACTION == 'Create'
                }
            }
            
            environment {     
                DOCKERHUB_CREDENTIALS = credentials('docker-cred')     
            }
            
            steps{
               sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin' 
               sh 'docker push $IMAGENAME'
               sh 'docker logout'
            }
        }
        
        /* 
        stage('Deploy Container: Docker'){
            
            when{
                expression{
                    params.ACTION == 'Create'
                }
            }
            
            environment {     
                DOCKERHUB_CREDENTIALS = credentials('docker-cred')     
            }
            
            steps{
               sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin' 
               sh 'docker pull $IMAGENAME'
               sh 'docker run -itd --name demo-app-$BUILD_NUMBER -p 5685:8080 $IMAGENAME'
               sh 'docker logout'
            }
        }
        */
    }
    
    post{
        always{
            cleanWs()
            sh 'docker container rm -f $(docker container ls -aq)'
            sh 'docker image rm -f $(docker image ls -aq)'
        }
    }
    
}
