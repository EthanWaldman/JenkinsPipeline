pipeline {
    agent any
    options { skipStagesAfterUnstable() }
    parameters {
	booleanParam(name: 'SKIPTESTS',
		defaultValue: false,
		description: 'Set to True to have pipeline bypass test stages')
    }
    
    stages {
        stage('Pull') {
            steps {
                git 'https://github.com/paulc4/microservices-demo'
            }
        }
        stage('Build') {
            steps {
                sh 'mvn compile'
            }
        }
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
	    when { expression { return !params.SKIPTESTS } }
        }
        stage('Package') {
                /* package goal will rerun the tests but keeping this structure as placeholder for explicit test runs */
            steps {
                sh 'mvn package'
		dir ("./target") {
			stash name: "jarfile", includes: "*.jar"
		}
            }
        }
        stage('Containerize') {
            steps {
		unstash name: "jarfile"
                sh '''
                    JAR_FILE=`ls microservice-demo-*.jar`
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
                    cat << EOI | sed 's/APPNAME/msdemo/' | sed 's/APPLABEL/msdemo/' | sed 's/JARNAME/microservice-demo/' > deployment.spec 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: APPNAME
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: APPLABEL
    spec:
      containers:
      - name: APPNAME
        image: 192.168.33.33:5000/JARNAME:latest
        ports:
        - containerPort: 8001
          hostPort: 81
EOI
                '''
                kubernetesDeploy configs: 'deployment.spec', kubeConfig: [path: '/var/lib/jenkins/kubeconfig'], secretName: '', ssh: [sshCredentialsId: '*', sshServer: ''], textCredentials: [certificateAuthorityData: '', clientCertificateData: '', clientKeyData: '', serverUrl: 'https://']
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

