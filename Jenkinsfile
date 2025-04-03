pipeline {
    agent any
    
    tools {
        maven 'Maven-3.6.3' // Tên tool Maven đã cấu hình trong Jenkins
        jdk 'jdk11' // Tên JDK đã cấu hình trong Jenkins
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                script {
                    // Xác định service nào thay đổi (sẽ triển khai ở phần 6)
                    def changedServices = detectChangedServices()
                    
                    if (changedServices.isEmpty()) {
                        echo "No service changes detected, building all services"
                        sh "mvn clean package -DskipTests"
                    } else {
                        echo "Building changed services: ${changedServices}"
                        changedServices.each { service ->
                            dir(service) {
                                sh "mvn clean package -DskipTests"
                            }
                        }
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    def changedServices = detectChangedServices()
                    
                    if (changedServices.isEmpty()) {
                        echo "No service changes detected, testing all services"
                        sh "mvn test"
                        junit '**/target/surefire-reports/*.xml'
                        jacoco execPattern: '**/target/jacoco.exec'
                    } else {
                        echo "Testing changed services: ${changedServices}"
                        changedServices.each { service ->
                            dir(service) {
                                sh "mvn test"
                                junit '**/target/surefire-reports/*.xml'
                                jacoco execPattern: '**/target/jacoco.exec'
                            }
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Upload test results và coverage
            junit '**/target/surefire-reports/*.xml'
            jacoco execPattern: '**/target/jacoco.exec'
            
            // Clean workspace
            cleanWs()
        }
    }
}

def detectChangedServices() {
    def changedServices = []
    
    // Lấy danh sách thay đổi từ Git
    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            def files = new ArrayList(entry.affectedFiles)
            for (int k = 0; k < files.size(); k++) {
                def file = files[k]
                def filePath = file.path
                
                // Xác định service từ path file thay đổi
                def service = determineServiceFromPath(filePath)
                if (service && !changedServices.contains(service)) {
                    changedServices.add(service)
                }
            }
        }
    }
    
    return changedServices
}

def determineServiceFromPath(String filePath) {
    // Danh sách các service trong project
    def services = [
        'api-gateway',
        'config-server',
        'customers-service',
        'discovery-server',
        'vets-service',
        'visits-service'
    ]
    
    // Kiểm tra file thuộc service nào
    for (service in services) {
        if (filePath.startsWith(service + '/')) {
            return service
        }
    }
    
    return null
<<<<<<< HEAD
}
=======
}
>>>>>>> 152414cf9e68975b6268349a7fc1242730224a7c
