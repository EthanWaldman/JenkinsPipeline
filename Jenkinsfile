pipeline {
    agent any
    tools { maven 'Default' }
    options { skipStagesAfterUnstable() }
    parameters {
	string(name: 'ServiceName', defaultValue: 'microservices-demo')
	booleanParam(name: 'SkipTests',
		defaultValue: false,
		description: 'Set to True to have pipeline bypass test stages')
    }
    
    stages {
        stage('Initialize') {
            steps {
                sh 'echo ${JOB_BASE_NAME} | cut -d"-" -f2- > .microservice_name'
            }
        }
        stage('Pull') {
            steps {
                sh 'env'
                git "https://github.com/EthanWaldman/${readFile('.microservice_name')}"
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
	    when { expression { return !params.SkipTests } }
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
            environment { SVCNAME = readFile('.microservice_name') }
            steps {
		unstash name: "jarfile"
                sh '''
                    JAR_FILE=`ls ${SVCNAME}-*.jar`
                    JAR_VERSION=`echo ${JAR_FILE} | sed 's/.jar$//' \
                         | sed "s/^${SVCNAME}-//"`
                    echo "${JAR_VERSION}" > .service_version
                    cat << EOI | sed "s/_JARNAME_/${JAR_FILE}/" > Dockerfile
FROM docker.io/labengine/centos
MAINTAINER Ethan ekwaldman@gmail.com
#
RUN yum install -y java-1.8.0-openjdk
ADD _JARNAME_ /
EXPOSE 8001
ENTRYPOINT java -jar /_JARNAME_ registration 8001
EOI
                    export DOCKER_HOST=192.168.33.33
                    docker build -t ${SVCNAME}:${JAR_VERSION} .
                    docker tag ${SVCNAME}:${JAR_VERSION} ${SVCNAME}:latest
                    docker tag ${SVCNAME}:${JAR_VERSION} 192.168.33.33:5000/${SVCNAME}:${JAR_VERSION}
                    docker push 192.168.33.33:5000/${SVCNAME}:${JAR_VERSION}
                '''
                stash name: "service-version", includes: ".service_version"
            }
        }
        stage('Deploy') {
            environment { SVCNAME = readFile('.microservice_name') }
            steps  {
		unstash name: "service-version"
		sh '''
			cat << EOI > kubeconfig
apiVersion: v1
clusters:
- cluster:
    server: http://192.168.33.41:8080
  name: lab
contexts:
- context:
    cluster: lab
    user: ""
  name: lab
current-context: lab
kind: Config
preferences: {}
users: []
EOI
		'''
                sh '''
		    SERVICE_VERSION=`cat ./.service_version`
                    cat << EOI | sed "s/_APPNAME_/${SVCNAME}/" | sed "s/_VERSION_/${SERVICE_VERSION}/" > deployment.spec 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: _APPNAME_
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: _APPNAME_
        service: _APPNAME_
        version: v_VERSION_
    spec:
      containers:
      - name: _APPNAME_
        image: 192.168.33.33:5000/_APPNAME_:_VERSION_
        ports:
        - containerPort: 8001
          hostPort: 81
EOI
                '''
		sh '''
#                        set +e
#			kubectl --kubeconfig ~/kubeconfig get deployment ${SVCNAME}
#			if [ $? -eq 0 ]
#			then
#				VERB=update
#			else
#				VERB=create
#			fi
#                        set -e
			kubectl --kubeconfig ./kubeconfig apply -f deployment.spec
		'''
            }
        }
        stage('Verify') {
            steps {
                timeout(time: 600, unit: 'SECONDS') {
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

