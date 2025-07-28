# Book My Show DevSecOps CI/CD Pipeline Documentation
## 1. Overview
This project demonstrates an end-to-end **DevSecOps pipeline** for a Book My Show clone. By seamlessly integrating Continuous Integration (CI), Continuous Delivery/Deployment (CD), and embedded security practices, the pipeline enables rapid, secure, and reliable releases. Key components include:

- **Source Code Management**: GitHub
- **CI/CD Orchestration**: Jenkins
- **Code Quality & Security Analysis**: SonarQube & Trivy
- **Containerization**: Docker
- **Orchestration & Deployment**: Kubernetes (AWS EKS)
- **Monitoring & Observability**: Prometheus & Grafana
---

## 2. Architecture Diagram
```plaintext
plaintextCopyEdit                +----------------------+
             |      GitHub Repo     |
             +----------+-----------+
                        |
                        v
             +----------------------+
             |       Jenkins        |
             |  (CI/CD Orchestration)|
             +------+----+----------+
                    |    |
           +--------+    +----------+
           |                       |
+---------------------+   +----------------------+
|   SonarQube         |   |      Trivy           |
| (Static Code & SAST)|   | (Container Scanning) |
+---------+-----------+   +-----------+----------+
          |                           |
          v                           v
       (Quality Gate)           (Vulnerability Scan)
          |                           |
          +------------+--------------+
                       |
                       v
              +---------------------+
              |  Docker Build &     |
              |  Push to Registry   |
              +---------+-----------+
                        |
                        v
              +---------------------+
              | AWS EKS (Kubernetes)|
              |  - Deployment Manifests  |
              +---------+-----------+
                        |
                        v
          +--------------------------+
          | Monitoring Infrastructure|
          |  - Prometheus & Grafana  |
          +--------------------------+
```
---

## 3. Components and Their Roles
### 3.1 GitHub
- **Role**: Hosts the source code.
- **Best Practice**: Use branch policies and pull request reviews to ensure quality.
### 3.2 Jenkins
- **Role**: Central orchestrator of the CI/CD workflow.
- **Key Aspects**:
    - Checkout code.
    - Trigger builds on commit.
    - Integrate with SonarQube, Trivy, and Docker.
    - Deploy to Kubernetes via shell scripts or Kubernetes plugins.
### 3.3 SonarQube
- **Role**: Performs **static code analysis (SAST)**.
- **Functions**:
    - Detects code smells, bugs, and potential vulnerabilities.
    - Enforces quality gates that block the pipeline if quality thresholds are not met.
- **DevSecOps Impact**: Introduces security early (shift-left testing) by ensuring code quality before any build artifacts are produced.
### 3.4 Trivy
- **Role**: Conducts vulnerability scanning on the built Docker images.
- **Functions**:
    - Scans for known Common Vulnerabilities and Exposures (CVEs).
    - Fails builds that contain critical or high-severity issues.
### 3.5 Docker
- **Role**: Containerizes the application ensuring consistency across environments.
- **Best Practice**: Optimize the Dockerfile for smaller images and use multi-stage builds where necessary.
### 3.6 Kubernetes (AWS EKS)
- **Role**: Manages container orchestration and scales the application.
- **Deployment**: Uses YAML manifests (e.g., deployment, service, ingress) to define how the application runs in production.
- **Scalability**: Facilitates rolling updates and high availability.
### 3.7 Prometheus & Grafana
- **Role**: Provide observability and monitoring.
- **Prometheus**:
    - Scrapes metrics from Node Exporter on cluster nodes and from Jenkins (if available).
    - Stores time-series data.
- **Grafana**:
    - Visualizes data with pre-built dashboards for system performance and pipeline metrics.
    - Helps in diagnosing performance issues and forecasting scalability.
---

## 4. Detailed Pipeline Workflow
### 4.1 Pipeline Stages
1. **Source Code Checkout**
    - **Action**: Jenkins polls the GitHub repository and checks out the code on new commits.
    - **Purpose**: Ensure the latest code is always used for the build.
2. **Static Analysis with SonarQube**
    - **Action**: The pipeline calls the `sonar-scanner`  tool.
    - **Parameters**: Project key, source location, SonarQube server URL, and authentication token.
    - **Outcome**: If the build fails the quality gate, the pipeline halts immediately.
3. **Security Scan with Trivy**
    - **Action**: Post-build, Trivy scans the generated Docker image.
    - **Outcome**: High or critical severity vulnerabilities cause the pipeline to fail.
4. **Docker Build & Push**
    - **Action**: Build a Docker image using the optimized Dockerfile.
    - **Push**: Upload the image to Docker Hub (or another container registry).
    - **Best Practices**: Tag images correctly and use caching strategies.
5. **Deployment to AWS EKS**
    - **Action**: Deploy the Docker container to a Kubernetes cluster using `kubectl apply -f k8s/` .
    - **Manifest Files**: Deployment, service, and ingress YAML files.
    - **Configuration**: Use environment variables and secrets to manage sensitive data.
6. **Monitoring & Observability**
    - **Setup**:
        - **Node Exporter** runs on all nodes, exposing metrics on port 9100.
        - **Prometheus** is configured with appropriate `scrape_configs`  to collect data.
        - **Grafana** displays dashboards (imported via JSON configuration or defined manually).
    - **Outcome**: Real-time insights into system health, pipeline performance, and potential issues.
### 4.2 Sample Jenkinsfile (Groovy)
```groovy
groovyCopyEditpipeline {
  agent any
  environment {
    SCANNER_HOME = tool 'sonar-scanner'
    DOCKER_IMAGE = 'yourusername/bms:latest'
    SONAR_TOKEN = credentials('sonar-token')
  }
  stages {
    stage('Checkout') {
      steps {
        git url: 'https://github.com/yourorg/BookMyShow.git', branch: 'main'
      }
    }
    stage('Code Analysis - SonarQube') {
      steps {
        withSonarQubeEnv('MySonarQubeServer') {
          sh """
            ${SCANNER_HOME}/bin/sonar-scanner \
            -Dsonar.projectKey=bookmyshow \
            -Dsonar.sources=. \
            -Dsonar.host.url=http://<sonar-url>:9000 \
            -Dsonar.login=${SONAR_TOKEN}
          """
        }
      }
    }
    stage('Security Scan - Trivy') {
      steps {
        sh 'trivy fs . --exit-code 1 --severity HIGH,CRITICAL'
      }
    }
    stage('Docker Build & Push') {
      steps {
        sh 'docker build -t $DOCKER_IMAGE .'
        withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
          sh 'echo $PASS | docker login -u $USER --password-stdin'
          sh 'docker push $DOCKER_IMAGE'
        }
      }
    }
    stage('Deploy to Kubernetes') {
      steps {
        sh 'kubectl apply -f k8s/'
      }
    }
  }
}
```
---

## 5. Monitoring Configuration
### 5.1 Prometheus
**Prometheus YAML Snippet:**

```yaml
yamlCopyEditglobal:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['<node1-ip>:9100', '<node2-ip>:9100']
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<jenkins-host>:8080']
```
- **Note**: Adjust target IPs/ports to match your environment.
### 5.2 Grafana
- **Installation**: Install Grafana on the monitoring server (e.g., Ubuntu via apt).
- **Access**: Default URL `http://<monitoring-VM-IP>:3000` 
- **Dashboards**: Import dashboards for Kubernetes, Node Exporter, and Jenkins performance.
- **Data Source**: Point Grafana to Prometheus (`http://<prometheus-server-ip>:9090` ).
---

## 6. Lessons Learned & Challenges
### What I Learned:
- Effective integration of security (SonarQube & Trivy) into the CI/CD workflow to implement shift-left strategies.
- The value of automating deployments to a Kubernetes cluster using AWS EKS.
- Setting up comprehensive monitoring (Prometheus & Grafana) for operational and pipeline insights.
### Challenges Faced:
- **Jenkins Agent Connectivity**: Configuring Jenkins in a dynamic Kubernetes environment required careful setup of agents and network policies.
- **Docker Image Optimization**: Balancing image size with build complexity.
- **SonarQube Quality Gates**: Fine-tuning thresholds to avoid false positives while ensuring code quality.
- **Monitoring Configurations**: Tuning Prometheus scrape intervals and Grafana dashboards to reflect meaningful metrics.
---

## 7. Future Enhancements
- **Auto-Rollbacks**: Implement mechanisms that automatically revert deployments if health checks fail.
- **GitOps Integration**: Adopt tools like ArgoCD for declarative continuous delivery.
- **Expanded Alerting**: Integrate Alertmanager with Prometheus to send notifications (email, Slack, etc.) based on defined thresholds.
- **Container-Level Metrics**: Enhance monitoring using additional tools like kube-state-metrics and the Prometheus Operator.
---

## 8. Conclusion
This documentation encapsulates the complete journey of setting up a robust DevSecOps pipeline—from code commit in GitHub to automated static analysis, secure image building, container orchestration on AWS EKS, and comprehensive monitoring. The integration of SonarQube and Trivy solidifies the security posture of the pipeline, ensuring that only high-quality, secure code reaches production. This project has been instrumental in understanding how modern DevOps practices evolve into a full DevSecOps lifecycle, ensuring faster, reliable, and more secure software deliveries.

