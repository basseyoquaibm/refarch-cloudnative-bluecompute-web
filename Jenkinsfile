podTemplate(label: 'mypod',
    volumes: [
        hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
        secretVolume(secretName: 'docker-account', mountPath: '/var/run/secrets/docker'),
        configMapVolume(configMapName: 'docker-repository', mountPath: '/var/run/configs/docker-config')
    ],
    containers: [
        containerTemplate(name: 'kubectl', image: 'lachlanevenson/k8s-kubectl', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'docker' , image: 'docker:17.06.1-ce', ttyEnabled: true, command: 'cat')
  ]) {

    node('mypod') {
        checkout scm
        container('docker') {
            stage('Build Docker Image') {
                sh """
                #!/bin/bash
                NAMESPACE=`cat /var/run/configs/docker-config/namespace`
                REGISTRY=`cat /var/run/configs/docker-config/registry`

                cp -ar StoreWebApp docker/StoreWebApp
                cd docker

                if [ \${REGISTRY} -eq "dockerhub" ]; then
                    # Docker Hub
                    docker build -t \${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER} .
                else
                    # Private Repository
                    docker build -t \${REGISTRY}/\${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER} .
                fi

                docker build -t \${REGISTRY}/\${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER} .
                rm -r StoreWebApp
                """
            }
            stage('Push Docker Image to Private Repository') {
                sh """
                #!/bin/bash
                NAMESPACE=`cat /var/run/configs/docker-config/namespace`
                REGISTRY=`cat /var/run/configs/docker-config/registry`

                set +x
                DOCKER_USER=`cat /var/run/secrets/docker/username`
                DOCKER_PASSWORD=`cat /var/run/secrets/docker/password`

                if [ \${REGISTRY} -eq "dockerhub" ]; then
                    # Docker Hub
                    docker login -u=\${DOCKER_USER} -p=\${DOCKER_PASSWORD}
                else
                    # Private Repository
                    docker login -u=\${DOCKER_USER} -p=\${DOCKER_PASSWORD} \${REGISTRY}
                fi
                set -x

                if [ \${REGISTRY} -eq "dockerhub" ]; then
                    # Docker Hub
                    docker push \${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER}
                else
                    # Private Repository
                    docker push \${REGISTRY}/\${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER}
                fi
                """
            }
        }
        container('kubectl') {
            stage('Deploy new Docker Image') {
                sh """
                #!/bin/bash
                set +e
                NAMESPACE=`cat /var/run/configs/docker-config/namespace`
                REGISTRY=`cat /var/run/configs/docker-config/registry`

                kubectl get deployments \${NAMESPACE}-bluecompute-ce-web

                if [ \${?} -eq "0" ]; then
                    # Update Deployment
                    kubectl set image deployment/\${NAMESPACE}-bluecompute-ce-web web=\${REGISTRY}/\${NAMESPACE}/bluecompute-ce-web:${env.BUILD_NUMBER}
                    kubectl rollout status deployment/\${NAMESPACE}-bluecompute-ce-web
                else
                    # No deployment to update
                    echo 'No deployment to update'
                    exit 1
                fi
                """
            }
        }
    }
}