pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                echo "Hello, Jenkins!"
           d}
        
    }
}pipeline {
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

// Hàm phát hiện service thay đổi (sẽ triển khai chi tiết ở phần 6)
def detectChangedServices() {
    def changedServices = []
    def changeLogSets = currentBuild.changeSets
    for (changeLog in changeLogSets) {
        for (entry in changeLog.items) {
            for (file in entry.affectedFiles) {
                def path = file.path
                if (path.contains('/')) {
                    def service = path.split('/')[0]
                    if (!changedServices.contains(service)) {
                        changedServices.add(service)
                    }
                }
            }
        }
    }
    return changedServices
}
