pipeline {
    agent any

    options {
        timeout(time: 1, unit: 'HOURS')
    }    

    stages {
        stage('Code CheckOut') {
            steps {
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
                    
                    // Copy artifact from S3 to PWD
                    withCredentials([usernamePassword(credentialsId: 'AWS-Credentials', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                        sh """
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            aws s3 cp ${s3_path} ${PWD} --debug
                        """
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
                    """
                    
                    // Calculate and compare SHA256 hashes directly
                    def calculatedModelHash = sh(script: "sha256sum \${PWD}/model_weights.json | awk '{print $1}'", returnStdout: true).trim()
                    def calculatedOutputHash = sh(script: "sha256sum \${PWD}/control_output.txt | awk '{print $1}'", returnStdout: true).trim()
                    
                    // Read checksums from checksum.txt
                    def checksumFile = readFile("${PWD}/checksum.txt")
                    def expectedModelHash = null
                    def expectedOutputHash = null
                    
                    checksumFile.readLines().each { line ->
                        if (line.startsWith("model_weights.json:")) {
                            expectedModelHash = line.split(":")[1].trim()
                        } else if (line.startsWith("control_output.txt:")) {
                            expectedOutputHash = line.split(":")[1].trim()
                        }
                    }
                    
                    // Debugging output
                    echo "Calculated Model Hash: ${calculatedModelHash}"
                    echo "Expected Model Hash: ${expectedModelHash}"
                    echo "Calculated Output Hash: ${calculatedOutputHash}"
                    echo "Expected Output Hash: ${expectedOutputHash}"
                    
                    // Compare hashes
                    if (calculatedModelHash == expectedModelHash && calculatedOutputHash == expectedOutputHash) {
                        echo "Hash verification successful!"
                        // Continue with further stages
                    } else {
                        error "Verification failed: Hash mismatch for files."
                    }
                }
            }
        }

        stage('Validation message') {
            steps {
                echo "Job executed as expected."
            }
        }
    }

    post {
        always{
            // Clean up workspace
            deleteDir()
        }
    }    
}
