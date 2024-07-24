pipeline {
    agent any

    options {
        timeout(time: 1, unit: 'HOURS')
    }

    stages {
        stage('Code Checkout') {
            steps {
                git branch: 'main', credentialsId: 'GitHub_Credentials', url: 'https://github.com/Singh-1991/Groovy-Mismatch-Values.git'
            }
        }

        stage('Get Artifact') {
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
                            aws s3 cp ${s3_path} ${PWD}/ --debug
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
                    
                    def checksumFile = "${PWD}/checksum.txt"
                    
                    // Check if checksum.txt exists
                    if (!fileExists(checksumFile)) {
                        error "Missing checksum file: ${checksumFile}"
                    }
                    
                    // Read checksum file contents
                    def checksumContent
                    try {
                        checksumContent = readFile(checksumFile)
                    } catch (Exception e) {
                        error "Failed to read checksum file: ${checksumFile}"
                    }
                    
                    // Validate checksum file format and hash values
                    def calculatedHashes = [:]
                    def missingHashes = []
                    
                    checksumContent.readLines().each { checksumLine ->
                        try {
                            def parts = checksumLine.split()
                            if (parts.size() >= 2) {
                                def expectedHash = parts[0].trim()
                                def filename = parts[1].trim()
                                
                                // Validate hash format
                                if (!expectedHash.matches(/[a-fA-F0-9]{64}/)) {
                                    error "Corrupted checksum file: ${checksumFile}"
                                }
                                
                                // Calculate hash for each file except checksum.txt itself
                                if (!filename.endsWith("checksum.txt")) {
                                    def fileToHash = "${PWD}/${filename}"
                                    if (!fileExists(fileToHash)) {
                                        error "Missing file: ${filename} expected in checksum file."
                                    }
                                    def calculatedHash = sh(script: "sha256sum ${fileToHash} | awk '{print \$1}'", returnStdout: true).trim()
                                    calculatedHashes[filename] = calculatedHash
                                    
                                    // Compare hashes
                                    if (expectedHash == calculatedHash) {
                                        echo "Hash verification successful for ${filename}!"
                                    } else {
                                        error "Verification failed: Hash mismatch for ${filename}. Expected: ${expectedHash}, Calculated: ${calculatedHash}"
                                    }
                                }
                            } else {
                                if (parts.size() == 0) {
                                    missingHashes.add("Missing hash in line: ${checksumLine}")
                                } else if (parts.size() == 1) {
                                    missingHashes.add("Missing hash for file in line: ${checksumLine}")
                                }
                            }
                        } catch (Exception ex) {
                            echo "Error processing line in checksum file: ${checksumLine}"
                            ex.printStackTrace()
                        }
                    }
                    
                    if (missingHashes) {
                        error "Missing hashes in checksum file: \n${missingHashes.join('\n')}"
                    }
                    
                    // Check for missing files in checksum file
                    def filesInChecksum = checksumContent.readLines().collect { line ->
                        line.split()[1].trim()
                    }.findAll { filename ->
                        !filename.endsWith("checksum.txt")
                    }
                    
                    def filesInWorkspace = sh(script: "ls ${PWD}", returnStdout: true).trim().split()

                    def missingFiles = filesInChecksum.findAll { filename ->
                        !filesInWorkspace.contains(filename)
                    }
                    
                    if (missingFiles) {
                        error "Missing files in workspace: ${missingFiles.join(', ')}"
                    }
                }
            }
        }

        stage('Validation Message') {
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
