pipeline {
    environment {
        IMAGE_REPO_NAME = "dev.docker.registry:5000/${(env.JOB_NAME.tokenize('/') as String[])[0]}"
        DR_IMAGE_REPO_NAME = "dr-registry.psw.gov.pk:5000/psw/${(env.JOB_NAME.tokenize('/') as String[])[0]}"
        REPO_NAME = "${scm.getUserRemoteConfigs()[0].getUrl().tokenize('/').last().split("\\.")[0]}" //Repository name - must be mapped in the service map dictionary below for CD
    }
    agent { label 'linux' }
    
    stages {
        stage('Figuring out the Committer') {
               steps {
                   echo "Figuring out the committer!"
                   script {
                         env.committerEmail = sh (
                           script: 'git --no-pager show -s --format=\'%ae\'',
                           returnStdout: true
                           ).trim()
                         println "Committer email is ${committerEmail}"
                   }
               }
        }
        stage('Get Dockerfile') {
            steps {
                dir("devops") {
                    //keeping devops dockerfiles in a subdirectory.
                    git credentialsId: 'git', url: 'https://git.psw.gov.pk/dev/devops.git'
                    sh "git checkout master"
                }
            }
        }
        stage('Dotnet Restore and Publish') {
            steps {
                //echo "Back on the original repository."
                //sh "ls"
                sh "dotnet clean"
                sh "dotnet restore"
                sh "dotnet publish -c Release -o out"
            }
        }
        stage('Docker Build') {
            environment {
                BUILD_IMAGE_REPO_TAG = "${env.IMAGE_REPO_NAME}:${env.BRANCH_NAME}-latest"
                DR_BUILD_IMAGE_REPO_TAG = "${env.DR_IMAGE_REPO_NAME}:${env.BRANCH_NAME}-latest"
            }
            steps {
                echo "test: ${(env.JOB_NAME.tokenize('/'))}"
                sh "docker build -f devops/dockerfiles/${(env.JOB_NAME.tokenize('/') as String[])[0]}/Dockerfile.Jenkins . -t $BUILD_IMAGE_REPO_TAG --network=host"
                sh "docker tag $BUILD_IMAGE_REPO_TAG ${env.IMAGE_REPO_NAME}:${env.BRANCH_NAME}_$BUILD_NUMBER"
                sh "docker tag $BUILD_IMAGE_REPO_TAG $DR_BUILD_IMAGE_REPO_TAG"
            }
        }
        stage('Docker Push') {
            environment {
                BUILD_IMAGE_REPO_TAG = "${env.IMAGE_REPO_NAME}:${env.BRANCH_NAME}-latest"
                DR_BUILD_IMAGE_REPO_TAG = "${env.DR_IMAGE_REPO_NAME}:${env.BRANCH_NAME}-latest"
            }
            steps {
                    sh "docker push $BUILD_IMAGE_REPO_TAG"
                    // sh "docker push $DR_BUILD_IMAGE_REPO_TAG"
                    sh "docker push ${env.IMAGE_REPO_NAME}:${env.BRANCH_NAME}_$BUILD_NUMBER"
            }
        }
        stage('SIT CD for Develop') {
            when {
                branch 'develop'    //This stage is only going to be executed for develop branch.
            }
            steps {
                    script {
                    // Since we use one Jenkinsfile for many pipelines and 1 repository is being utilized in more than one CI pipeline.
                    // We can use a map/dictionary to determine which repository holds two (or more) CI pipelines, for deployment, both the relevant services will be deployed on relevant server.
                    // Webapp, mock, landing_app (no webhooks, no gitflow) and extgateway have their own Jenkinsfile in their repositories.
                    SERVICE_MAP = [ "wfs" : "wfs,wfew,",
                        "adi" : "adi,adiw,",
                        "ans" : "ans,answ,",
                        "apigateway" : "apigateway,",
                        "auth" : "auth,authw,",
                        "clm" : "clm,clmw,",
                        "fss" : "fss,",
                        "irms" : "irms,", 
                        "oga" : "oga,",   //ogaw has been opted out for now.
                        "misapi" : "misapi,",
                        "sd" : "sd,sdw,",
                        "shrd" : "shrd,",
                        "upsapp" : "upsapp,",
                        "tarp" : "tarp,",
                        "psi" : "psi,",
                        "lms" : "lms,",
                        "itt" : "itt,",
                        "ups" : "ups,upsw,"
                    ]
                    println SERVICE_MAP."$REPO_NAME"
                }
                echo "CI was successful. Triggering deployment on SIT for develop branches!"
                echo "The branch name is ${env.BRANCH_NAME}"
                //echo "The repository name is: ${env.REPO_NAME} and the service being triggered are ${SERVICE_MAP["$env.REPO_NAME"]}"
                //the parameter 'Services' is defined in the remote pipeline.
                build job: 'On-Demand-SIT-Deployment', parameters: [string(name: 'Services', value: SERVICE_MAP."$REPO_NAME")]
            }
        }
    }
    post {
        //In case of a entirely successful CI/CD cycle, the deployment email will be sent by the remote CD Job.
        //In case the CI fails, an email will be sent out to the committer + DevOps.
        failure {
            echo "The committer email was: ${env.committerEmail}"
            emailext to: "${env.committerEmail}, DevOps@psw.gov.pk", from: 'no-reply-jenkins@psw.gov.pk',
                subject: "[Jenkins] Build failure email for ${env.JOB_NAME}",
                body: "Job failed - \"${env.JOB_NAME}\" build: ${env.BUILD_NUMBER}\n\nView the log at:\n ${env.BUILD_URL}"
        }
    }
}