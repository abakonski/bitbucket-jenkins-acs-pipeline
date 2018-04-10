import java.text.SimpleDateFormat

node {
    try {
        // Run builds for this job in the 'myapp' sub-directory within Jenkins' workspace path
        dir('myapp') {
            // Get start time of this job - we'll measure the duration
            def measureStartDT = new Date()

            // Image names should be in the format <ACR_URL>/<IMAGE_LABEL>:<IMAGE_VERSION>
            // This will allow them to be pushed and pulled from ACR correctly
            def imageName = 'myacregistry.azurecr.io/myapp'
            def imageNameTagged = imageName + ':' + env.BUILD_NUMBER

            // BitBucket repository URL
            def bbRepositoryUrl = 'ssh://bitbucket.org/myteam/myapp.git'

            // ACR login server URL  
            def acrUrl = 'myacregistry.azurecr.io'

            // ACS login details
            // TIP: To find the ACS DNS, open the Deployments section for your ACS in Azure portal, then look at the Output section
            // TIP: To find the username for your ACS, look at the Input section for your Deployment and look for LINUXADMINUSERNAME
            def acsMasterDNS = 'myapp-staging-master.centralus.cloudapp.azure.com'
            def acsUsername = 'swarmuser'
            

            // First stage - pull the 'develop' branch from BitBucket
            stage('Checkout develop branch from BitBucket') {
                // Slack notification about current progress
                slackSend color: 'warning', message: '[myapp-staging] Checking out staging code from BitBucket'
                
                // Pull the code from BitBucket
                git branch: 'develop', credentialsId: 'bitbucket-login', url: bbRepositoryUrl
            }

            // Second stage - clean out previous Docker images from Jenkins server
            stage('Clean out previous Docker images from Jenkins server') {
                // Slack notification about current progress
                slackSend color: 'warning', message: '[myapp-staging] Cleaning out previous Docker images from Jenkins server'

                // Remove all except latest image for this project
                result = sh(returnStdout: true, script: "docker images -aq ${imageName}").trim()
                sh 'echo "${result}"'
                if (result != '') {
                    imageDeleteMarker = sh(returnStdout: true, script: "docker images -aq -f before=${imageName}:\$(docker images -a ${imageName} | awk '{print \$2}' | egrep -x '[0-9]+' | sort -rn | head -n1) ${imageName}").trim()
                    if (imageDeleteMarker != '') {
                        sh "docker image rm -f ${imageDeleteMarker}"
                    }
                }
            }
        
            // Third stage - build and push Docker image to Azure Container Registry... 
            stage('Build and push Docker image to ACR') {
                // Slack notification about current progress
                slackSend color: 'warning', message: '[myapp-staging] Building and pushing Docker image to ACR'

                // Bake correct image tag into docker-compose.yml file
                // NOTE: Locally we use a different image tag, so we just need to replace it here
                sh "sed -i -e 's|image: 127.0.0.1:5000/myapp:latest|image: ${imageNameTagged}|g' docker-compose.yml"

                // Build the Docker image
                docker.build(imageNameTagged)

                // Log into ACR using SPN credentials, then push the image to ACR
                withCredentials([usernamePassword(credentialsId: 'acr-login', passwordVariable: 'spn_pw', usernameVariable: 'spn_uname')]) {
                    sh "docker login ${acrUrl} -u ${spn_uname} -p ${spn_pw}"
                    sh "docker push ${imageNameTagged}"
                }
            }

            // Fourth stage - connect to ACS, pull image from ACR, deploy to stack
            stage('ACS Docker Pull and Run - Central US') {
                // Slack notification about current progress
                slackSend color: 'warning', message: '[myapp-staging] Connecting to ACS cluster'

                // Open SSH Tunnel to ACS Cluster
                sshagent(['acs-login']) {
                    // TIP: If this Jenkins server deploys to more than one cluster, assign a different local port for each
                    sh "ssh -fNL 2375:localhost:2375 -p 2200 ${acsUsername}@${acsMasterDNS} -o StrictHostKeyChecking=no -o ServerAliveInterval=240 && echo 'ACS SSH Tunnel successfully opened...'"
                }

                // Set env variable for stage for SSH ACS Tunnel
                withCredentials([usernamePassword(credentialsId: 'acr-login', passwordVariable: 'spn_pw', usernameVariable: 'spn_uname')]) {
                    // Set 'DOCKER_HOST' environment variable to local port assigned above so commands are run on the remote Docker host, not locally
                    env.DOCKER_HOST=':2375'

                    // Print out debug info
                    sh 'echo "DOCKER_HOST is $DOCKER_HOST"'
                    sh 'docker info'

                    // Log in to ACR
                    sh "docker login ${acrUrl} -u ${spn_uname} -p ${spn_pw}"

                    // Slack notification about current progress
                    slackSend color: 'warning', message: '[myapp-staging] Pulling image from ACR and running stack deploy'

                    // Pull the Docker image from ACR
                    sh "docker pull ${imageNameTagged}"
                    //sh "docker service update --force iqboxy-hub_redis"

                    // Slack notification about current progress
                    slackSend color: 'warning', message: '[myapp-staging] Deploying stack to ACS'

                    // Deploy the stack to ACS cluster
                    sh "docker stack deploy --with-registry-auth --compose-file docker-compose.yml myapp-staging"
                }

                // Slack notification about current progress
                slackSend color: 'good', message: '[myapp-staging] Deployment complete 💃🏻🕺'
                slackSend color: 'good', message: '[myapp-staging] Deployment took ' + ((new Date().getTime() - measureStartDT.getTime())/1000) + ' seconds. @channel please test!'
            }
        }
    }
    catch (Exception err) {
        // Slack notification about ERROR encountered during deployment
        slackSend color: '#f00', message: '[myapp-staging] Deployment failed! @channel please debug: (' + err.message + ')'
        
        throw err
    }
}