pipeline {
    agent any

    options {
        timeout(time: 1, unit: 'HOURS')
    }    

    stages {
        stage('Code CheckOut') {
            steps{
                git branch: 'main', credentialsId: 'GitHub_Credentials', url: 'https://github.com/Singh-1991/wnames-script.git'
            }
        }     
        
        stage('Get artifact') {
            steps {
              script {
                def versionsManifest = readYaml file: 'versions_manifest.yml'
                def s3_path = versionsManifest.version_info.ML_model.cloud_model.path
                def tarball_name = s3_path.tokenize('/')[-1]
                // Ensure PWD is correctly defined within the container
                def PWD = sh(script: "echo \$(pwd)", returnStdout: true).trim()
                // env.PWD = PWD
                // Copy artifact from S3 to PWD
                withCredentials([usernamePassword(credentialsId: 'AWS-Credentials', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    sh """
                        export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                        export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                    """
                    sh "aws s3 cp ${s3_path} ${PWD} --debug"
                }
              }
            }
        }

        stage('Validate Hash') {
            steps {
                script {
                    def versionsManifest = readYaml file: 'versions_manifest.yml'
                    def s3_path = versionsManifest.version_info.ML_model.cloud_model.path
                    def tarball_name = s3_path.tokenize('/')[-1]                        
                    // Ensure PWD is correctly defined within the container
                    def PWD = sh(script: "echo \$(pwd)", returnStdout: true).trim()
                    
                    // Untar the tarball and calculate sha256 hashes
                    sh """
                        tar -xvf ${PWD}/${tarball_name} -C ${PWD}
                        cd ${PWD}
                        sha256sum model_weights.json > model_weights.json.sha256
                        sha256sum control_output.txt > control_output.txt.sha256
                    """
                        
                    // Read checksums from checksum.txt
                    def checksumFile = readFile("${PWD}/checksum.txt")
                    def checksumLines = checksumFile.readLines()
                    
                    // Process checksums
                    def expectedModelHash = null
                    def expectedOutputHash = null
                        
                    checksumLines.each { line ->
                        if (line.startsWith("model_weights.json:")) {
                            expectedModelHash = line.split(":")[1].trim()
                        } else if (line.startsWith("control_output.txt:")) {
                            expectedOutputHash = line.split(":")[1].trim()
                        }
                    }
                        
                    // Read calculated hashes
                    def actualModelHash = sh(script: "cat ${PWD}/model_weights.json.sha256 | cut -d' ' -f1", returnStdout: true).trim()
                    def actualOutputHash = sh(script: "cat ${PWD}/control_output.txt.sha256 | cut -d' ' -f1", returnStdout: true).trim()
                        
                    // Compare hashes
                    if (actualModelHash == expectedModelHash && actualOutputHash == expectedOutputHash) {
                        echo "Hash verification successful!"
                        // Continue with further stages
                    } else {
                        error "Verification failed: Hash mismatch for files."
                    }
                }
            }
    }

        stage('Validation message'){
            steps {
                sh "echo Job Executed As Expected"
            }
        }
    }
}  
