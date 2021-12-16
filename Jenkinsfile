pipeline {

    environment {
        IMAGE_NAME = "webapp"
        IMAGE_TAG = "1.1"
        STAGING = "daniel-ajc-staging-env"
        PRODUCTION = "daniel-ajc-prod-env"
        USERNAME = "lianhuahayu"
        CONTAINER_NAME = "webapp"
        EC2_PRODUCTION_HOST = "54.88.166.62"
    }

    agent none

    stages{

       stage ('Build Image'){
           agent any
           steps {
               script{
                   sh '''
                        docker build -t $USERNAME/$IMAGE_NAME:$IMAGE_TAG .
                   '''
               }
           }
       }

        stage ('clean env and save artifact') {
           agent any
           environment{
               PASSWORD = credentials('dockerhub_password')
           }
           steps {
               script{
                   sh '''
                       docker login -u $USERNAME -p $PASSWORD
                       docker push $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                       docker stop $CONTAINER_NAME || true
                       docker rm $CONTAINER_NAME || true
                       docker rmi $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                   '''
               }
           }
       }

        
        stage('Deploy app on EC2-cloud Staging') {
            agent any
            when{
               expression{ GIT_BRANCH == 'origin/master'}
            }
            steps{
                withCredentials([sshUserPrivateKey(credentialsId: "ec2_prod_private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        script{ 

                            //timeout(time: 15, unit: "MINUTES") {
                            //    input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                            //}

                            sh'''
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker stop ${CONTAINER_NAME}-staging || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker rm ${CONTAINER_NAME}-staging || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker run --name ${CONTAINER_NAME}-staging -d -p 8088:80 $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                            '''
                        }
                    }
                }
            }
        }

        stage ('Test application staging') {
           agent any
           steps {
               script{
                   sh '''
                       test=`curl -Is http://localhost:8088 | head -n 1`
                       if [ test == 'HTTP/1.1 200 OK' ]; then true; else false; fi

                   '''
               }
           }
       }

        stage('Deploy app on EC2-cloud Production') {
            agent any
            when{
               expression{ GIT_BRANCH == 'origin/master'}
            }
            steps{
                withCredentials([sshUserPrivateKey(credentialsId: "ec2_prod_private_key", keyFileVariable: 'keyfile', usernameVariable: 'NUSER')]) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        script{ 

                            timeout(time: 15, unit: "MINUTES") {
                                input message: 'Do you want to approve the deploy in production?', ok: 'Yes'
                            }

                            sh'''
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker stop ${CONTAINER_NAME}-prod || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker rm ${CONTAINER_NAME}-prod || true
                                ssh -o StrictHostKeyChecking=no -i ${keyfile} ${NUSER}@${EC2_PRODUCTION_HOST} docker run --name ${CONTAINER_NAME}-prod -d -p 8090:80 $USERNAME/$IMAGE_NAME:$IMAGE_TAG
                            '''
                        }
                    }
                }
            }
        }

        stage ('Test application prod') {
           agent any
           steps {
               script{
                   sh '''
                       test=`curl -Is http://localhost:8090 | head -n 1`
                       if [ test == 'HTTP/1.1 200 OK' ]; then true; else false; fi
                   '''
               }
           }
       }

    }

    post {
        success{
            slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
        failure {
            slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
        }
    }

}
