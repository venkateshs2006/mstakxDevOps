def project = 'mstakx'
def  appName = 'guestbook'
def  feSvcName = "${appName}-frontend"
def  imageTag = "192.168.0.245:5000/guestbook"
 stages {
    stage('Test') {
      steps {
        container(${imageTag}) {
          sh """
            curl -e  10.10.40.93:30100 # HaProxy Server URL with NodePort number
          """
        }
      }
    }
    stage('Build and push image with Container Builder') {
      steps {
        container('guestbook') {
          sh "docker builds submit -t ${imageTag} ."
          sh "docker push ${imageTag} ."
        }
      }
    }
    stage('Deploy Production') {
      // Production branch
      when { branch 'master' }
      steps{
        container('kubectl') {
        // Change deployed image in canary to the one we just built
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/production/*.yaml")
          sh("kubectl --namespace=production apply -f k8s/services/")
          sh("kubectl --namespace=production apply -f k8s/production/")
          sh("echo http://`kubectl --namespace=production get service/${feSvcName} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'` > ${feSvcName}")
        }
      }
    }
    stage('Deploy stage') {
      // Developer Branches
      when { 
        not { branch 'master' } 
      } 
      steps {
        container('kubectl') {
          // Create namespace if it doesn't exist
          sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
          // Don't use public load balancing for development branches
          sh("sed -i.bak 's#LoadBalancer#ClusterIP#' ./k8s/services/frontend.yaml")
          sh("sed -i.bak 's#gcr.io/cloud-solutions-images/gceme:1.0.0#${imageTag}#' ./k8s/stage/*.yaml")
          sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/services/")
          sh("kubectl --namespace=${env.BRANCH_NAME} apply -f k8s/stage/")
         }
      }     
    }
  }
}
