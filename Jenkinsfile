pipeline {
    agent any
    environment {
        CHANGED_FILES = sh(script: "git diff --name-only HEAD^ HEAD", returnStdout: true).trim()
    }
    stages {
        stage('Detect Changes') {
            steps {
                script {
                    def services = ['customers-service', 'vets-service', 'visits-service']
                    def triggeredServices = []
                    
                    for (service in services) {
                        if (CHANGED_FILES.contains(service + "/")) {
                            triggeredServices.add(service)
                        }
                    }
                    
                    if (triggeredServices.isEmpty()) {
                        currentBuild.result = 'ABORTED'
                        error("No relevant changes detected. Skipping build.")
                    } else {
                        env.TRIGGERED_SERVICES = triggeredServices.join(',')
                    }
                }
            }
        }
        
        stage('Test & Build') {
            steps {
                script {
                    def services = env.TRIGGERED_SERVICES.split(',')
                    for (service in services) {
                        dir(service) {
                            sh './mvnw test'
                            sh './mvnw package'
                        }
                    }
                }
            }
        }
    }
}
