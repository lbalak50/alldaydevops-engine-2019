def imageName = 'imdb-engine'
def bucket = 'add-deployment-packages'
def function = 'ADDiMDBEngine'
def region = 'eu-central-1'

node('slaves'){
    try {
        stage('Checkout'){
            checkout scm
        }

        stage('Quality Test'){
            docker.build("${imageName}:${env.BUILD_ID}", '-f Dockerfile.quality .')
            sh "docker run --rm ${imageName}:${env.BUILD_ID}"
        }

        stage('Unit Test'){
            docker.build("${imageName}:${env.BUILD_ID}", '-f Dockerfile.unit .')
            sh "docker run --rm ${imageName}:${env.BUILD_ID}"
        }

        stage('Security Test'){
            docker.build("${imageName}:${env.BUILD_ID}", '-f Dockerfile.security .')
            sh "docker run --rm ${imageName}:${env.BUILD_ID}"
        }

        stage('Build'){
            docker.build(imageName)
        }

        stage('Push'){
            sh """
                docker run -d --name ${imageName} ${imageName}
                docker cp ${imageName}:/root/app main
                zip ${commitID()}.zip main
                aws s3 cp ${commitID()}.zip s3://add-deployment-packages/
            """
        }

        stage('Deploy'){
            sh "aws lambda update-function-code --function-name ${function} --s3-bucket ${bucket} --s3-key ${commitID()}.zip --region ${region}"

            if(env.BRANCH_NAME == 'master'){
                def version = sh (
                    script: "aws lambda publish-version --function-name ${function} --description production-${commitID()} --region ${region} | jq -r '.Version'",
                    returnStdout: true
                ).trim()
                sh "aws lambda update-alias --function-name ${function} --name production --function-version ${version} --region ${region}"
            }
            
            if(env.BRANCH_NAME == 'preprod'){
                def version = sh (
                    script: "aws lambda publish-version --function-name ${function} --description staging-${commitID()} --region ${region} | jq -r '.Version'",
                    returnStdout: true
                ).trim()
                sh "aws lambda update-alias --function-name ${function} --name staging --function-version ${version} --region ${region}"
            }

            if(env.BRANCH_NAME == 'develop'){
                def version = sh (
                    script: "aws lambda publish-version --function-name ${function} --description sandbox-${commitID()} --region ${region} | jq -r '.Version'",
                    returnStdout: true
                ).trim()
                sh "aws lambda update-alias --function-name ${function} --name sandbox --function-version ${version} --region ${region}"
            }
        }
    } finally{
        sh "docker rm -f ${imageName}"
    }
}

def commitID() {
    sh 'git rev-parse HEAD > .git/commitID'
    def commitID = readFile('.git/commitID').trim()
    sh 'rm .git/commitID'
    commitID
}