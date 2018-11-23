podTemplate(
    label: 'mypod', 
    inheritFrom: 'default',
          containerTemplate(
            name: 'helm', 
            image: 'ibmcom/k8s-helm:v2.6.0',
            ttyEnabled: true,
            command: 'cat'
        )
  
    node('mypod') {
            
        stage ('Deploy') {
            checkout scm
            container ('helm') {
                sh "/helm init --client-only --skip-refresh"
                sh "/helm upgrade post post/charts/post --install --kube-context=minikube --set image.tag=latest"
            }
        }
    }
 
