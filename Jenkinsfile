node {
    // def project = 'k8s-workshop-203112'
    def appName = 'gceme'
    def feSvcName = "${appName}-frontend"
    def username = "testuserwsk8s"
    // def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
    def imageTag = "${username}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"
    
    checkout scm

    stage('Preparation') {
        sh 'curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.9.7/bin/linux/amd64/kubectl'
        sh 'chmod +x ./kubectl && mv kubectl /usr/local/sbin'
        sh 'curl -LO https://github.com/stedolan/jq/releases/download/jq-1.5/jq-linux64'
        sh 'chmod +x ./jq-linux64 && mv jq-linux64 /usr/local/sbin/jq'
    }

    stage('Build image') {
        sh("docker build -t ${imageTag} .")
    }

    stage('Run Go tests') {
        sh("docker run ${imageTag} go test")
    }

    stage('Push image to registry') {
        sh("docker login -u testuserwsk8s -p cacamaca32")
        sh("docker push ${imageTag}")
    }

    stage("Deploy Application") {
        switch (env.BRANCH_NAME) {
            // Roll out to canary environment
            case "canary":
                // Change deployed image in canary to the one we just built
                sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
                sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/canary/*.yaml")
                sh("kubectl --namespace=production apply -f k8s/services/")
                sh("kubectl --namespace=production apply -f k8s/canary/")
                sh("sleep 4") 
                sh ("echo http://`kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'`")
                break

            // Roll out to production
            case "master":
                // Change deployed image in canary to the one we just built
                sh("kubectl get ns production || kubectl create ns production")
                sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/production/*.yaml")
                sh("kubectl --namespace=production apply -f k8s/services/")
                sh("kubectl --namespace=production apply -f k8s/production/")
                sh("sleep 4")
                sh("echo http://`kubectl --namespace=production get service/${feSvcName} --output=json | jq -r '.status.loadBalancer.ingress[0].ip'`")
                break

            // Roll out a dev environment
            default:
                // Create namespace if it doesn't exist
                sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
                // Don't use public load balancing for development branches
                sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
                sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/dev/*.yaml")
                sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/services/")
                sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/dev/")
                echo 'To access your environment run `kubectl proxy`'
                echo "Then access your service via http://localhost:8001/api/v1/proxy/namespaces/${env.BRANCH_NAME}/services/${feSvcName}:80/"
        }
    }
}