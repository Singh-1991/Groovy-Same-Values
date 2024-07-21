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
                    def calculatedHashes = [:]
                    def checksumFile = readFile("${PWD}/checksum.txt")
                    
                    // Read each line in checksum file and store hash values
                    checksumFile.readLines().each { checksumLine ->
                        def parts = checksumLine.split()
                        if (parts.size() >= 2) {
                            def expectedHash = null
                            def filename = parts[1].trim()
                            
                            // Calculate hash for each file except checksum.txt itself
                            if (!filename.endsWith("checksum.txt")) {
                                def calculatedHash = sh(script: "sha256sum ${PWD}/${filename} | awk '{print \$1}'", returnStdout: true).trim()
                                calculatedHashes[filename] = calculatedHash
                                
                                // Debugging output
                                echo "Calculated Hash for ${filename}: ${calculatedHash}"
                                
                                // Find expected hash for the current file
                                checksumFile.each { line ->
                                    if (line.startsWith("${filename}:")) {
                                        expectedHash = line.split(":")[1].trim()
                                    }
                                }
                                
                                // Compare hashes
                                if (expectedHash && expectedHash == calculatedHash) {
                                    echo "Hash verification successful for ${filename}!"
                                } else {
                                    error "Verification failed: Hash mismatch for ${filename}. Expected: ${expectedHash}, Calculated: ${calculatedHash}"
                                }
                            }
                        }
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
