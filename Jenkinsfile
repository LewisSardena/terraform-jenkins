pipeline {
    agent any
    tools {
        "org.jenkinsci.plugins.terraform.TerraformInstallation" "terraform-0.11.7"
    }
    parameters {
        string(name: 'LAMBDA_URL', defaultValue: '', description: 'URL to the Lamdba function')
        string(name: 'WORKSPACE', defaultValue: 'development', description:'worspace to use in Terraform')
    }
    environment {
        TF_HOME = tool('terraform-0.11.7')
        TF_IN_AUTOMATION = "true"
        PATH = "$TF_HOME:$PATH"
        DYNAMODB_STATELOCK = "ddt-tfstatelock"
        NETWORKING_BUCKET = "ddt-networkingsardena"
        NETWORKING_ACCESS_KEY = credentials('networking_access_key')
        NETWORKING_SECRET_KEY = credentials('networking_secret_key')
        APPLICATION_BUCKET = "ddt-applicationsardena"
        APPLICATION_ACCESS_KEY = credentials('application_access_key')
        APPLICATION_SECRET_KEY = credentials('application_secret_key')
    }
    stages {
        stage('NetworkInit'){
            steps {
                dir('networking/'){
                    cmd 'terraform --version'
                    cmd "terraform init -input=false -plugin-dir=/var/jenkins_home/terraform_plugins \
                     --backend-config='dynamodb_table=$DYNAMODB_STATELOCK' --backend-config='bucket=$NETWORKING_BUCKET' \
                     --backend-config='access_key=$NETWORKING_ACCESS_KEY' --backend-config='secret_key=$NETWORKING_SECRET_KEY'"
                    cmd "echo \$PWD"
                    cmd "whoami"
                }
            }
        }
        stage('NetworkPlan'){
            steps {
                dir('networking/'){
                    script {
                        try {
                           cmd "terraform workspace new ${params.WORKSPACE}"
                        } catch (err) {
                            cmd "terraform workspace select ${params.WORKSPACE}"
                        }
                        cmd "terraform plan -var 'aws_access_key=$NETWORKING_ACCESS_KEY' -var 'aws_secret_key=$NETWORKING_SECRET_KEY' \
                        -var 'url=${params.LAMBDA_URL}' -out terraform-networking.tfplan;echo \$? > status"
                        stash name: "terraform-networking-plan", includes: "terraform-networking.tfplan"
                    }
                }
            }
        }
        stage('NetworkApply'){
            steps {
                script{
                    def apply = false
                    try {
                        input message: 'confirm apply', ok: 'Apply Config'
                        apply = true
                    } catch (err) {
                        apply = false
                        bat "terraform destroy -var 'aws_access_key=$NETWORKING_ACCESS_KEY' -var 'aws_secret_key=$NETWORKING_SECRET_KEY' -force"
                        currentBuild.result = 'UNSTABLE'
                    }
                    if(apply){
                        dir('networking/'){
                            unstash "terraform-networking-plan"
                            bat 'terraform apply terraform-networking.tfplan'
                        }
                    }
                }
            }
        }
        stage('ApplicationInit'){
            steps {
                dir('applications/'){
                    bat "terraform init -input=false -plugin-dir=/var/jenkins_home/terraform_plugins \
                     --backend-config='dynamodb_table=$DYNAMODB_STATELOCK' --backend-config='bucket=$APPLICATION_BUCKET' \
                     --backend-config='access_key=$APPLICATION_ACCESS_KEY' --backend-config='secret_key=$APPLICATION_SECRET_KEY'"
                    bat "echo \$PWD"
                    bat "whoami"
                }
            }
        }
        stage('ApplicationPlan'){
            steps {
                dir('applications/'){
                    script {
                        try {
                            bat "terraform workspace new ${params.WORKSPACE}"
                        } catch (err) {
                            bat "terraform workspace select ${params.WORKSPACE}"
                        }
                        bat "terraform plan -var 'aws_access_key=$APPLICATION_ACCESS_KEY' -var 'aws_secret_key=$APPLICATION_SECRET_KEY' \
                        -var 'url=${params.LAMBDA_URL}' -out terraform-application.tfplan;echo \$? > status"
                        stash name: "terraform-application-plan", includes: "terraform-application.tfplan"
                    }
                }
            }
        }
        stage('ApplicationApply'){
            steps {
                script{
                    def apply = false
                    try {
                        input message: 'confirm apply', ok: 'Apply Config'
                        apply = true
                    } catch (err) {
                        apply = false
                        bat "terraform destroy -var 'aws_access_key=$APPLICATION_ACCESS_KEY' -var 'aws_secret_key=$APPLICATION_SECRET_KEY' -force"
                        currentBuild.result = 'UNSTABLE'
                    }
                    if(apply){
                        dir('applications/'){
                            unstash "terraform-application-plan"
                            bat 'terraform apply terraform-application.tfplan'
                        }
                    }
                }
            }
        }
    }
}