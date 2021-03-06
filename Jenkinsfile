pipeline {

agent {
        kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  name: nodejs-deployment
  labels:
    some-label: test-odu
    label : jenkins
spec:
  securityContext:
    runAsUser: 10000
    runAsGroup: 10000
  containers:
  - name: jnlp
    image: 'jenkins/jnlp-slave:4.3-4'
    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
  - name: yair
    image: us.icr.io/dc-tools/security/yair:1
    command:
    - cat
    tty: true
    imagePullPolicy: Always
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug-1534f90c9330d40486136b2997e7972a79a69baf
    imagePullPolicy: Always
    command:
    - cat
    tty: true
    volumeMounts:
      - name: regsecret
        mountPath: /kaniko/.docker
    securityContext: # https://github.com/GoogleContainerTools/kaniko/issues/681
      runAsUser: 0
      runAsGroup: 0
  - name: openshift-cli
    image: openshift/origin-cli:v3.11.0
    command:
    - cat
    tty: true
    securityContext: # https://github.com/GoogleContainerTools/kaniko/issues/681
      runAsUser: 0
      runAsGroup: 0
  volumes:
  - name: regsecret
    projected:
      sources:
      - secret:
          name: regsecret
          items:
            - key: .dockerconfigjson
              path: config.json
  imagePullSecrets:
  - name: oduregsecret
  - name: regsecret
"""
        }
    }
  environment {
    
     /* -----------DevOps Commander  created env variables------------ */
	
	DOCKER_URL= "us.icr.io/dc-tools"
DOCKER_CREDENTIAL_ID= "dc-docker-6110"
OPENSHIFT_URL= "https://c100-e.us-south.containers.cloud.ibm.com:32634/"
OPENSHIFT_CREDENTIAL_ID= "dc-openshift-6110"
SONARQUBE_URL= "https://sonarqube-3-6.container-crush-02-4044f3a4e314f4bcb433696c70d13be9-0000.eu-de.containers.appdomain.cloud/"
SONARQUBE_CREDENTIAL_ID= "dc-sonarqube-6110"
CLAIR_URL= "https://clair-3-3.container-crush-02-4044f3a4e314f4bcb433696c70d13be9-0000.eu-de.containers.appdomain.cloud/"
CLAIR_CREDENTIAL_ID= "dc-clair-6110"
NAMESPACE= "tools-dev"
INGRESS= "isdc20-0ce42e8480356580312b8efcc5f21aad-0001.us-south.containers.appdomain.cloud"
	
	/* -----------DevOps Commander  created env variables------------ */

          VERSION="1-SNAPSHOT"

        /* Parameters for Docker image that will be built and deployed */
        /* REGISTRY_USERNAME provided via a Jenkins secret
         * REGISTRY_PASSWORD provided via a Jenkins secret
         */
        REGISTRY_NAME="us.icr.io/dc-tools"
	//REGISTRY_NAME="odureg.azurecr.io"
        REGISTRY_SECRET="odu-registry"
        DOCKER_IMAGE="lnk"
        DOCKER_TAG="$BUILD_NUMBER"
	K8S_DEPLOYMENT="nodejs-test"
	POD_NAME ="nodejs_deployment"
       
  }
 
  stages {
    
    stage ('Build: Maven') {
            steps {
                withMaven(
                    maven: 'maven-3',
           //        mavenSettingsConfig: 'java-dc',
                    mavenLocalRepo: '.repository'
                ) {
                    sh 'mvn -Dmaven.test.failure.ignore=true clean package'
                }
            }
        }

        stage ('Build: Docker') {
            steps {
                container('kaniko') {
                    /* Kaniko uses secret 'regsecret' declared in the POD to authenticate to the registry and push the image */
                    sh 'pwd && ls -l && df -h && cat /kaniko/.docker/config.json && /kaniko/executor -f `pwd`/Dockerfile --context `pwd` --insecure --skip-tls-verify --cache=true --destination=${REGISTRY_NAME}/${DOCKER_IMAGE}:${DOCKER_TAG}'
                }
            }
        }
        stage ('Secure: Image scan - Clair') {
            steps {
                container('yair') {
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIAL_ID}", usernameVariable: 'REGISTRY_USERNAME', passwordVariable: 'REGISTRY_PASSWORD')]) {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') { /* Avoid the stage being FAILURE if Clair founds vulnerabilities above acceptable threshold. That will be calculated later in the gate */
                              //  sh 'yair.py --clair ${CLAIR_URL} --registry ${REGISTRY_NAME} --username ${REGISTRY_USERNAME} --password ${REGISTRY_PASSWORD} --no-namespace ${DOCKER_IMAGE}:${DOCKER_TAG}'
				 sh 'wget https://github.com/optiopay/klar/releases/download/v2.4.0/klar-2.4.0-linux-amd64'
				 sh 'mv klar-2.4.0-linux-amd64 klar'
				 sh 'chmod 755 klar'
		                 sh 'export CLAIR_ADDR=${CLAIR_URL} && export DOCKER_PASSWORD=${REGISTRY_PASSWORD} && export DOCKER_USER=${REGISTRY_USERNAME} && ./klar ${DOCKER_URL}/${DOCKER_IMAGE}:${DOCKER_TAG}  '
                            }
                        }
                    }
                }
            }
            post {
                always {
                    sh 'rm -rf $WORKSPACE/reports/clair && mkdir -p $WORKSPACE/reports/clair'
           //         sh 'cp clair-results.json $WORKSPACE/reports/clair'
                }
            }
        }		
	
	 stage('Deploy: To Openshift') {
        steps {
          container('openshift-cli') {
	     withCredentials([
	        usernamePassword(credentialsId: "${OPENSHIFT_CREDENTIAL_ID}", usernameVariable: 'REGISTRY_USERNAME', passwordVariable: 'TOKEN'),
		      usernamePassword(credentialsId: "${DOCKER_CREDENTIAL_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')
	     ]) {
              sh '''
              oc login --server="${OPENSHIFT_URL}" --token="${TOKEN}"
              oc project ${NAMESPACE}
              pwd
              ls -ltr
              oc create secret docker-registry docker-repo-cred \
              --docker-server=${DOCKER_URL} \
              --docker-username=${DOCKER_USERNAME} \
              --docker-password=${DOCKER_PASSWORD} \
              --docker-email=${DOCKER_PASSWORD} \
              --namespace=${NAMESPACE} \
              || true
              sed -e "s~{REGISTRY_NAME}~$DOCKER_URL~g" \
                  -e "s~{DOCKER_IMAGE}~$DOCKER_IMAGE~g" \
                  -e "s~{DOCKER_TAG}~$DOCKER_TAG~g" \
                  -e "s~{K8S_DEPLOYMENT}~$K8S_DEPLOYMENT~g" \
		  -e "s~{NAMESPACE}~$NAMESPACE~g"\
                  -e "s~{INGRESS_URL}~$INGRESS~g" -i devops/k8s/*.yml
              oc apply -f devops/k8s/ --namespace="${NAMESPACE}" \
              || true
	      oc expose svc ${K8S_DEPLOYMENT}-svctriala || true
              oc get route ${K8S_DEPLOYMENT}-svctriala
              oc wait --for=condition=available --timeout=120s deployment/${K8S_DEPLOYMENT} --namespace="${NAMESPACE}" \
              || true
              '''
	     }
           }
         }
        }

        


    }
}

