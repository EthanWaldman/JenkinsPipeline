pipeline {
    agent any
    options { skipStagesAfterUnstable() }
    
    stages {
        stage('Pull') {
            steps {
                git 'https://github.com/paulc4/microservices-demo'
            }
        }
        stage('Build') {
            steps {
/*                sh 'mvn compile' */
                  sh '''
			echo "Forced failure!"
			exit 1
                  '''
            }
        }
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                /* package goal will rerun the tests but keeping this structure as placeholder for explicit test runs */
                success {
                    sh 'mvn package'
                }
            }
        }
        stage('Store Artifact') {
            steps {
                    sh 'cp target/*.jar /var/tmp/localrepo'
            }
        }
        stage('Containerize') {
            steps {
                sh '''
                    cp /var/tmp/localrepo/microservice-demo-* .
                    JAR_FILE=`ls microservice-demo-*`
                    cat << EOI | sed "s/JARNAME/${JAR_FILE}/" > Dockerfile
FROM docker.io/labengine/centos
MAINTAINER Ethan ekwaldman@gmail.com
#
RUN yum install -y java-1.8.0-openjdk
ADD JARNAME /
EXPOSE 8001
ENTRYPOINT java -jar /JARNAME registration 8001
EOI
                    docker build -t microservice-demo:latest .
                    docker tag microservice-demo:latest 192.168.33.33:5000/microservice-demo:latest
                    docker push 192.168.33.33:5000/microservice-demo:latest
                '''
            }
        }
        stage('Deploy') {
            steps  {
                sh '''
                    cat << EOI | sed 's/PODNAME/msdemo/' | sed 's/PODLABEL/msdemo/' | sed 's/JARNAME/microservice-demo/' > deploypod 
apiVersion: v1
kind: Pod
metadata:
  name: PODNAME
  labels:
    name: PODLABEL
spec:
  containers:
    - name: PODNAME
      image: 192.168.33.33:5000/JARNAME:latest
      ports:
        - containerPort: 8001
          hostPort: 81
EOI
                '''
                kubernetesDeploy configs: 'deploypod', kubeConfig: [path: '/var/lib/jenkins/kubeconfig'], secretName: '', ssh: [sshCredentialsId: '*', sshServer: ''], textCredentials: [certificateAuthorityData: '', clientCertificateData: '', clientKeyData: '', serverUrl: 'https://']
            }
        }
        stage('Verify') {
            steps {
                timeout(time: 300, unit: 'SECONDS') {
                    sh '''
                        STATUS=-1
                        while [ ${STATUS} -ne 0 ]
                        do
                            sleep 1
                            set +e
                            curl 192.168.33.41:81
                            STATUS=$?
                            set -e
                        done
                    '''
                }
            }
        }
    }
}

