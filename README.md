# Maven-Powered Web Application CI/CD Pipeline

REFERENCE : https://github.com/spring-projects/spring-petclinic.git 

![image](https://github.com/user-attachments/assets/daa35ee6-f6d6-4699-8660-17014367830b)


# Example: Spring pet clinic CI CD pipeline



Comprehensive CI/CD Pipeline Explanation



Tools required: Jenkins ,Git, Sonarqube server, Docker ,Kubernetes , Grafana and Promotheus.
1.	Source Code Management:

•	Automated Repository Cloning: Jenkins is meticulously configured to clone our GitHub repository, with the SCM (Source Code Management) plugin set to poll for changes daily. This ensures our pipeline is always working with the latest codebase.

2.	Code Quality and Security:

•	SonarQube Integration for Robust Quality Checks: Once the repository is cloned, Jenkins triggers a comprehensive code quality and security analysis using SonarQube. This integration not only checks for code quality but also identifies potential security vulnerabilities, ensuring our code adheres to the highest standards. Detailed reports are generated and stored on the SonarQube server for continuous review.

3.	Efficient Build Process:

•	Artifact Generation: Our build system compiles the source code, generating a .jar file as the build artifact. This file is then strategically copied to the Docker context directory.
•	Optimized Docker Image Creation: The Dockerfile is designed to include everything from the directory where it resides. By copying the .jar file into the Docker image during the build process, we ensure that the application is ready to run within the container, making it accessible on port 8080 upon execution.

4.	Versioned Docker Image Management:

•	Image Tagging with Git Commit ID: To maintain clear traceability, each Docker image is tagged with the corresponding Git commit ID. This practice allows us to easily identify and track the source code version used for any specific image.
•	Seamless Push to Docker Hub: The pipeline then pushes the newly created Docker image to our Docker Hub repository, ensuring the latest version is readily available for deployment.

5.	Kubernetes Deployment and Configuration:

•	Dynamic Image Update: We pass the new Docker image name (tagged with the Git commit ID) as an environment variable, updating the pod deployment YAML file dynamically.
•	Automated Deployment: By applying the updated deployment YAML file and the NodePort balancer service configuration, our pipeline ensures that the latest Docker image is deployed across the Kubernetes cluster. This guarantees that our application is always running the latest build.

6.	Verification and Continuous Monitoring:

•	Comprehensive Checks: Post-deployment, the pipeline verifies that all pods are running successfully and the NodePort service is operational. This step ensures the deployment integrity and service availability.
•	Advanced Monitoring with Grafana and Prometheus: Our Kubernetes cluster, including critical nodes like knode1, is continuously monitored using Grafana and Prometheus. These tools provide real-time insights into system performance, application health, and resource utilization.
•	Quality Assurance through SonarQube Reports: In parallel, SonarQube's detailed reports confirm that our code quality metrics are met, providing an additional layer of assurance.

Workflow Summary
1.	Automated Repository Management: Jenkins clones the GitHub repository daily, ensuring up-to-date code.
2.	Rigorous Quality Assurance: SonarQube performs in-depth code quality and security analysis.
3.	Efficient Build and Deployment:
		o	Code is built, and a .jar file is generated and copied to the Docker context.
		o	Dockerfile includes all contents from its directory, creating a Docker image tagged with the Git commit ID.
		o	Docker image is pushed to Docker Hub for availability.
4.	Dynamic Kubernetes Deployment:
		o	Environment variables update the deployment YAML file with the new Docker image.
		o	Deployment and NodePort configurations are applied to Kubernetes.
5.	Verification and Monitoring:
		o	Pods and services are verified for successful operation.
		o	Continuous monitoring with Grafana and Prometheus.
		o	SonarQube ensures code quality is maintained.

Outcome
•	Robust Application Deployment: Our application is seamlessly deployed, running in the Kubernetes cluster, and accessible via the Node Port service on port 8080.
•	Continuous Monitoring: Ensures the health, performance, and reliability of our application and infrastructure.
•	Quality Assurance: SonarQube confirms code quality, reinforcing our commitment to delivering high-quality, secure software.
This pipeline represents a robust, automated system that enhances efficiency, maintains high standards of code quality and security, and ensures seamless, reliable deployment and monitoring of our applications.



 
SCM configured for every minute repo changes checking
 
groovy Script of spring pet CI CD: 

pipeline {

    agent {
        node {
            label 'kmaster'
        }
    }
		
    stages {
        stage('SCM Checkout') {
            steps {
                dir('/home/kmaster/testdir') {
                    git branch: 'main', url: 'https://github.com/Supreetp755/spring-petclinic.git'
                }
            }
        }
				
        stage('Build and SonarQube Analysis') {
            steps {
                dir('/home/kmaster/testdir') {
                    withSonarQubeEnv('Sonar') {
                        sh './mvnw clean package verify sonar:sonar -DskipTests'
                        script {
                            GIT_COMMIT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                            echo "GIT_COMMIT=${GIT_COMMIT}"
                        }
                    }
                }
            }
        }
        stage('Prepare Docker Build') {
            steps {
                script {
                    sh """
                        cd /home/kmaster/dockerbuild
                        rm -rf *
                        cp /home/kmaster/spring-petclinic_dockerfolder/Dockerfile /home/kmaster/dockerbuild/
                        cp /home/kmaster/testdir/target/spring-petclinic-3.3.0-SNAPSHOT.jar /home/kmaster/dockerbuild/
                    """
                }
            }
        }
        stage('Docker Build and Push') {
            steps {
                script {
                    def IMAGE_NAME = "springpetsonarci:${GIT_COMMIT}"
                    sh """
                        cd /home/kmaster/dockerbuild
                        docker build -t ${IMAGE_NAME} .
                        docker tag ${IMAGE_NAME} dockersupreet555/dockerspringpetsonarci:${GIT_COMMIT}
                        echo "##### push image to repository #####"
                        docker push dockersupreet555/dockerspringpetsonarci:${GIT_COMMIT}
                    """
                }
            }
        }
        stage('Deployment of pods through k8s Cluster') {
            steps {
                script {
                    def DOCKER_IMAGE = "dockersupreet555/dockerspringpetsonarci:${GIT_COMMIT}"
                    sh """
                        cd /home/kmaster/dockerbuild
                        DOCKER_IMAGE="${DOCKER_IMAGE}"
                        cp /home/kmaster/springpetdeploymentpod.yaml /home/kmaster/dockerbuild
                        cp /home/kmaster/springpetdeploymentsvc.yaml /home/kmaster/dockerbuild
                        cd /home/kmaster/testdir  // Change to the testdir for Kubernetes deployment
                        // Replace the placeholder in the YAML file with the actual Docker image
                        sed -i 's|IMAGE_PLACEHOLDER|${DOCKER_IMAGE}|g' /home/kmaster/dockerbuild/springpetdeploymentpod.yaml
                        kubectl apply -f /home/kmaster/dockerbuild/springpetdeploymentsvc.yaml -f /home/kmaster/dockerbuild/springpetdeploymentpod.yaml
                        kubectl get pods -o wide
                        kubectl get svc -o wide
                    """
                }
            }
        }
    }
    post {
        always {
            echo 'Pipeline finished.'
        }
    }
}



DockerFile:
![image](https://github.com/user-attachments/assets/9c3eb2c5-a691-4a25-aa54-8a3e4773cb68)

 
K8s pods deployment yaml file:
 
![image](https://github.com/user-attachments/assets/84cdbef7-55b3-49ea-99d9-90cbade5cf27)

K8s svc yaml file:

 ![image](https://github.com/user-attachments/assets/28c1e7c6-67d8-4763-93e0-2e9c13cab464)



CI CD Pipeline Ran successfully :

  ![image](https://github.com/user-attachments/assets/518e8e8f-fb16-4f26-83c6-11ff7bb73e15)
	![image](https://github.com/user-attachments/assets/5baae7e2-af1a-49b0-8366-8a9ebca94b64)



Console output :               
![image](https://github.com/user-attachments/assets/6cd8f8bc-88b6-426d-b7f7-190b6ade1fcb)

Web application is running successfully in nodeport assigned port 

![image](https://github.com/user-attachments/assets/4bdf4947-0f54-4ac8-b4c8-b52eea02fad2)

In docker repo as image is pushed recently ( 8 min before)  

![image](https://github.com/user-attachments/assets/83c3b5c2-c964-4cdf-b672-6001bd9da750)



Sonar cube report :

   ![image](https://github.com/user-attachments/assets/3cfe3b3a-87e9-4e73-9c54-a6b7cdfecbc6)


K8s Cluster pod deployment analysis:
 
![image](https://github.com/user-attachments/assets/d22963fc-9f74-4290-bed0-6242793661cd)



As pipeline started at 17:40 so u can observer spike in the below screenshots the VM health  is displayed in detail.
    
![image](https://github.com/user-attachments/assets/69688a4a-c691-4ae0-b48f-b41866aa7f79)


Node Health is displayed once deployment is started at time 17:41 and continued           
![image](https://github.com/user-attachments/assets/fab5c3d1-b914-465f-85c2-ad8bb67692ad)
![image](https://github.com/user-attachments/assets/58730130-ffee-4337-acef-ee5d2765469e)



