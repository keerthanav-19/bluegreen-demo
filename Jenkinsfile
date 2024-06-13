def label = "mypod-${UUID.randomUUID().toString()}"
def serviceaccount = "jenkins-admin"
def DOCKER_HUB_ACCOUNT_NAME = 'keerthanav123'
def DOCKER_IMAGE_NAME = 'testapp'
def IMAGE_TAG = "${BUILD_ID}"
def IMAGE_TAG_INPUT = "${BUILD_ID}"
def K8S_DEPLOYMENT_NAME = 'testapp'
def kubectl_image = 'smesch/kubectl'

podTemplate(label: label, serviceAccount: serviceaccount, containers: [
    containerTemplate(name: 'kubectl', image: kubectl_image, ttyEnabled: true, command: 'cat', volumes: [secretVolume(secretName: 'kube-config', mountPath: '/root/.kube')])])
{
    node(label){
        stage('Git Checkout')
        {
            sh '''
				git init
				git clone https://github.com/Keerthanachinnu/bluegreen-demo.git
				cd bluegreen-demo
				git checkout main
				ls -lart
            '''

        }  
        stage('Create and Push Image')
        {
            withCredentials([usernamePassword(credentialsId: 'docker-creds', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                sh '''
					cd bluegreen-demo
					pwd	
                    docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}
                    docker build -t ${DOCKER_USERNAME}/${DOCKER_IMG_NAME}:${IMAGE_TAG}
                    docker push ${DOCKER_USERNAME}/${DOCKER_IMG_NAME}:${IMAGE_TAG}
                '''
            }
        }
        stage('Deploy Build to Kubernetes Test Environment') 
        {
            container('kubectl')
            {
                sh '''
                    kubectl get svc > svc.txt
                    if grep -q '''+K8S_DEPLOYMENT_NAME+''' svc.txt; then
				              echo "Kubernetes service already exists" >> status.txt
				            else
              	      echo "No Kubernetes Service is found. It is a new environment "
              	      echo " Deployed in a fresh Enviroment" >> log.txt
				            fi
                    
                '''
                if (fileExists('status.txt'))
              {
              sh '''
              #!/bin/bash
              cd bluegreen-demo
              ROLE=$(kubectl get svc '''+K8S_DEPLOYMENT_NAME+''' -o jsonpath='{.spec.selector.role}')
              apk add gettext
              export K8S_DEPLOYMENT_NAME='''+K8S_DEPLOYMENT_NAME+'''
              export DOCKER_IMAGE_NAME='''+DOCKER_IMAGE_NAME+'''
              export IMG_NAME=$DOCKER_HUB_ACCOUNT_NAME$DOCKER_IMAGE_NAME
              export IMG_TAG='''+IMAGE_TAG_INPUT+'''
              export REPLICAS=1              
              if [ $ROLE == "green" ]
              then
                  export TARGET_ROLE=blue              
                  echo $TARGET_ROLE > role.txt
                  envsubst < '''+K8S_DEPLOYMENT_NAME+'''.yaml | kubectl apply -f -
                  
              elif [ $ROLE == "blue" ]
              then
                  export TARGET_ROLE=green
                  echo $TARGET_ROLE > role.txt
                  envsubst < '''+K8S_DEPLOYMENT_NAME+'''.yaml | kubectl apply -f -                
              fi
              '''	  				
        
              }
        
              else 
              {
              sh '''
              cd bluegreen-demo
              apk add gettext
              export K8S_DEPLOYMENT_NAME='''+K8S_DEPLOYMENT_NAME+'''
              export DOCKER_IMAGE_NAME='''+DOCKER_IMAGE_NAME+'''
              export IMG_NAME=$DOCKER_HUB_ACCOUNT_NAME$DOCKER_IMAGE_NAME
              export IMG_TAG='''+IMAGE_TAG_INPUT+'''
              export REPLICAS=1
              export TARGET_ROLE=blue
              envsubst < '''+K8S_DEPLOYMENT_NAME+'''-service.yaml | kubectl apply -f -
              envsubst < '''+K8S_DEPLOYMENT_NAME+'''.yaml | kubectl apply -f -
              echo $TARGET_ROLE > role.txt
              '''

              }

            }
        }

        stage('Deploy to kubernetes Environment  - Live Environment')
        {
            container('kubectl')
            {
              if (fileExists('status.txt'))
        {
              sh '''
              #!/bin/bash
              cd bluegreen-demo
              ROLE=$(kubectl get svc '''+K8S_DEPLOYMENT_NAME+''' -o jsonpath='{.spec.selector.role}')
              if [ $ROLE == "green" ]
              then                      
                kubectl patch svc '''+K8S_DEPLOYMENT_NAME+''' -p '{\"spec\":{\"selector\": {\"role\": \"blue\"}}}'
                echo "........Making Blue Environment as the Live URL......." >> log.txt
              elif [ $ROLE == "blue" ]
              then
                kubectl patch svc '''+K8S_DEPLOYMENT_NAME+''' -p '{\"spec\":{\"selector\": {\"role\": \"green\"}}}'
                echo ".....Making Green Environment as the Live URL....." >> log.txt
              fi
              echo ".....URL to access the application....." >> log.txt
              ELB=$(kubectl get svc '''+K8S_DEPLOYMENT_NAME+''' | awk '{print $4}' | grep -v EXTERNAL-IP)
              echo "http://"$ELB >> log.txt 
              '''
        }      
        else {
          sh'''
          cd bluegreen-demo
          echo ".........Initial Deployment .. Keeping Blue as Live environment......" >> log.txt
          echo ".....URL to access the application....." >> log.txt
          ELB=$(kubectl get svc '''+K8S_DEPLOYMENT_NAME+''' | awk '{print $4}' | grep -v EXTERNAL-IP)
          echo "http://"$ELB >> log.txt
          ''' 
        }      

        }

        }
    }
}