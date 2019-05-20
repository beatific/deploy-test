#!groovy
podTemplate(label: 'deploy-test', containers: [
    containerTemplate(name: 'kubectl', image: 'smesch/kubectl', ttyEnabled: true, command: 'cat',
        volumes: [secretVolume(secretName: 'kube-config', namespace: 'ns-jenkins', mountPath: '/root/.kube')]),
    containerTemplate(name: 'maven', image: 'maven', ttyEnabled: true, command: 'cat'),
    containerTemplate(name: 'docker', image: 'docker', ttyEnabled: true, command: 'cat',
        envVars: [containerEnvVar(key: 'DOCKER_CONFIG', value: '/tmp/'),])],
        volumes: [secretVolume(secretName: 'docker-config', namespace: 'ns-jenkins', mountPath: '/tmp'),
                  hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
  ]) {

    node('deploy-test') {

        properties(
            [
                    [$class: 'ParametersDefinitionProperty', parameterDefinitions:
                            [
                                    [$class: 'StringParameterDefinition', defaultValue: 'development', description: 'Maven에서 Active 할 Profile 을 입력하세요. 예) production', name: 'activeProfile']
                            ]
                    ]])
        def DOCKER_HUB_ACCOUNT = 'beatific'
        def DOCKER_IMAGE_NAME = 'deploy-test'
        def K8S_DEPLOYMENT_NAME = 'deploy-test'
        def POD_NAMESPACE = 'default'
        
        def mvnHome
        
        stage('Preparation') { // for display purposes
          echo "Current workspace : ${workspace}"
          // Get the Maven tool.
          // ** NOTE: This 'M3' Maven tool must be configured
          // **       in the global configuration.
          mvnHome = tool 'M3'
        }

        stage('Clone test-webapp-1 App Repository') {
            checkout scm
            
            
            container('maven') {
              stage('Build') {
                  sh "'${mvnHome}/bin/mvn' -P ${activeProfile} -Dmaven.test.skip=true clean install"
              }
            }
                        
            container('docker') {
                stage('Docker Build & Push Current & Latest Versions') {
                    sh ("docker build -t ${DOCKER_HUB_ACCOUNT}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} -f ./Dockerfile ./")
                    sh ("docker push ${DOCKER_HUB_ACCOUNT}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}")
                    sh ("docker tag ${DOCKER_HUB_ACCOUNT}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ${DOCKER_HUB_ACCOUNT}/${DOCKER_IMAGE_NAME}:latest")
                    sh ("docker push ${DOCKER_HUB_ACCOUNT}/${DOCKER_IMAGE_NAME}:latest")
                }
            }

            container('kubectl') {
                stage('Deploy New Build To Kubernetes') {
                    sh ("kubectl set image -n ${POD_NAMESPACE} deployment/${K8S_DEPLOYMENT_NAME} ${K8S_DEPLOYMENT_NAME}=${DOCKER_HUB_ACCOUNT}/${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}")
                }
            }

        }        
    }
}