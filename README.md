# Jenkins node script

 node('master') {
 
    // git variables
    
    git_url='git@github.com:vivekhimalayas/node-js.git'
    git_credentials='xxxxxxxxxxxxxxx'
    //docker variables
    docker_registry_url='144117137830.dkr.ecr.us-east-2.amazonaws.com'
    env.docker_registry='144117137830.dkr.ecr.us-east-2.amazonaws.com/'
    docker_repo='144117137830.dkr.ecr.us-east-2.amazonaws.com/node:v1'
    env.imagename="node"
    env.region="us-east-2"
    //rollback=null
    //ecs variables
    env.task_family="node"
    env.service_name="nodejs"
    env.cluster_name="node"
   
   stage('Clean WorkSpace'){
   
       sh 'rm -rf *'
   }
   stage('Git CheckOut') {
   
      // Get some code from a GitHub repository
        dir('code') {
          git branch: '$branch_name' , credentialsId: git_credentials, url: git_url
        }
     
   }
   

   stage('Build Docker Image') {
   
     sh"""
     cd node-js/
    docker build . --tag \${docker_registry}\${imagename}:\${BUILD_NUMBER}
     """
   }
   
   // Login and push image to ECR
   
   stage('Push Image to Us-east-2 ECR'){
   
     sh '$(aws ecr get-login --no-include-email --region $region)'
     
     sh"""
     docker push \${docker_registry}\${imagename}:\${BUILD_NUMBER}
     """
       
   }
   
   stage('Delete image from local'){
   
       sh"""
     docker rmi -f \${docker_registry}\${imagename}:\${BUILD_NUMBER}
     """
   }
   
   
   stage('ECS Deployment'){

    env.new_docker_image=docker_repo+":"+env.BUILD_NUMBER

        sh """

            TASK_DEF_OLD=\$(aws ecs describe-task-definition --task-definition  \$task_family --region \$region)
            TASK_DEF_NEW=\$(echo \$TASK_DEF_OLD | jq --arg NDI \$new_docker_image '.taskDefinition.containerDefinitions[0].image=\$NDI')
            TASK_FINAL=\$(echo \$TASK_DEF_NEW | jq '.taskDefinition|{family: .family, volumes: .volumes, containerDefinitions: .containerDefinitions}')
            FINAL_TASK_FOR_ROLLBACK=\$(echo \$TASK_DEF_OLD | jq '.taskDefinition|{family: .family, volumes: .volumes, containerDefinitions: .containerDefinitions}')
            echo -n \$FINAL_TASK_FOR_ROLLBACK > FINAL_TASK_FOR_ROLLBACK
            aws ecs register-task-definition --family \$task_family --region \$region --cli-input-json "\$(echo \$TASK_FINAL)"
            aws ecs update-service --service \$service_name --region \$region --task-definition \$task_family --cluster \$cluster_name

        """
    }
   
    stage('Verifying Deployment'){
        println "Waiting service to reached steady-state"
        try{
            sh "aws ecs wait services-stable --region \$region --cluster \$cluster_name --services \$service_name"
            rollback=false
        }
        catch (Exception e){
            //deployment failed, do rollback
            rollback=true
        }
        finally{
            //leaving it empty for future use
   
        }
    }

    if (rollback){
          stage ('rollback deployment'){
              println "Ready for deployment roll back"
              sh """
              aws ecs register-task-definition --region \$region --family \$task_family --cli-input-json "\$(cat FINAL_TASK_FOR_ROLLBACK)"
              aws ecs update-service --region \$region --service \$service_name --task-definition \$task_family --cluster \$cluster_name
              """
          }
      }
   
    if (rollback){
        error 'Deployment failed: Had to rollback'
    }

}
