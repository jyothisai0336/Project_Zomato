<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:FF4B2B,100:FF416C&height=220&section=header&text=Project%20Zomato&fontSize=60&fontColor=ffffff&fontAlignY=35&desc=A%20DevSecOps%20Pipeline%20That%20Blocks%20Vulnerable%20Code%20Before%20It%20Ships&descSize=18&descAlignY=55&animation=fadeIn" width="100%"/>

<br/>

<a href="#-architecture">Architecture</a> •
<a href="#-pipeline-flow">Pipeline</a> •
<a href="#-security-gates">Security</a> •
<a href="#-infrastructure">Infra</a> •
<a href="#-quick-start">Quick Start</a> •
<a href="#-contact">Contact</a>

<br/>

![Build](https://img.shields.io/badge/build-passing-success?style=for-the-badge&logo=jenkins&logoColor=white)
![Quality Gate](https://img.shields.io/badge/sonarqube-passed-brightgreen?style=for-the-badge&logo=sonarqube&logoColor=white)
![Security](https://img.shields.io/badge/trivy-0%20critical-success?style=for-the-badge&logo=aqua&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue?style=for-the-badge)

![GitHub last commit](https://img.shields.io/github/last-commit/jyothisai0336/Project_Zomato?style=flat-square&color=FF416C)
![GitHub repo size](https://img.shields.io/github/repo-size/jyothisai0336/Project_Zomato?style=flat-square&color=FF416C)
![GitHub stars](https://img.shields.io/github/stars/jyothisai0336/Project_Zomato?style=flat-square&color=FF416C)

</div>

<br/>

## 📖 Overview

**Project Zomato** is a Dockerized reference deployment of a Zomato-style food discovery application, shipped end-to-end through a fully automated **DevSecOps pipeline**. Every commit is built, statically analyzed, scanned for vulnerabilities, and deployed across a **multi-node Docker Swarm cluster on AWS EC2** — and the pipeline physically cannot push an image to the registry if the code doesn't pass quality and security gates first.

This isn't a "Docker run and pray" deployment. It's gated, observable, and repeatable.

> 💡 **The core idea:** security and quality checks aren't a report you read after the fact — they're a wall the pipeline can't get through if the code fails.

<br/>

## 🏗️ Architecture

```mermaid
flowchart LR
    A[👨‍💻 Developer Push] --> B[📦 GitHub]
    B --> C[⚙️ Jenkins Pipeline]
    C --> D[🔍 SonarQube CQA]
    D --> E{Quality Gate}
    E -->|❌ Fail| X[🛑 Pipeline Halted]
    E -->|✅ Pass| F[🐳 Docker Build]
    F --> G[🛡️ Trivy Image Scan]
    G --> H{Vulnerabilities?}
    H -->|❌ Found| X
    H -->|✅ Clean| I[📤 Push to Registry]
    I --> J[🚀 docker stack deploy]
    J --> K[🖥️ Swarm Manager]
    K --> L[Worker Node 1]
    K --> M[Worker Node 2]

    style E fill:#FF416C,stroke:#333,color:#fff
    style H fill:#FF416C,stroke:#333,color:#fff
    style X fill:#1a1a1a,stroke:#FF416C,color:#fff
    style I fill:#22c55e,stroke:#333,color:#fff
    style J fill:#22c55e,stroke:#333,color:#fff
```

<br/>

## ⚙️ Pipeline Flow

A **10-stage Jenkins Declarative Pipeline** takes every commit from raw code to a live, multi-node deployment — averaging **under 6 minutes** end to end.

| # | Stage | What Happens |
|---|-------|---------------|
| 1 | **Tool Install** | Provisions required CLI tools for the build agent |
| 2 | **Clean Workspace** | Wipes the workspace for a reproducible build |
| 3 | **Code** | Pulls the latest commit from GitHub |
| 4 | **CQA** | Static code analysis via SonarQube |
| 5 | **Quality Gates** | 🚦 Hard stop — pipeline halts if SonarQube flags don't clear |
| 6 | **Build** | Application build |
| 7 | **Image** | Docker image build |
| 8 | **Image Scan** | 🛡️ Trivy scans the image for CVEs before it goes anywhere |
| 9 | **Push** | Clean, scanned image pushed to the registry |
| 10 | **Deploy** | `docker stack deploy` rolls the release out across the Swarm |

<details>
<summary><b>🔍 Click to see the actual Jenkinsfile structure</b></summary>

```groovy
pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }

        stage('Code') {
            steps {
                git branch: 'main', url: 'https://github.com/jyothisai0336/Project_Zomato.git'
            }
        }

        stage('CQA - SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato
                    '''
                }
            }
        }

        stage('Quality Gates') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t zomato:${BUILD_NUMBER} .'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image zomato:${BUILD_NUMBER} --severity HIGH,CRITICAL --exit-code 1'
            }
        }

        stage('Push to Registry') {
            steps {
                sh 'docker push <registry>/zomato:${BUILD_NUMBER}'
            }
        }

        stage('Deploy to Swarm') {
            steps {
                sh 'docker stack deploy -c docker-compose.yml zomato-stack'
            }
        }
    }
}
```

</details>

<br/>

## 🛡️ Security Gates

<table>
<tr>
<td width="50%" valign="top">

### SonarQube — Quality Gate

```
✅ Status:          PASSED
🐛 New Bugs:         0
🔓 New Vulnerabilities: 0
🔥 New Security Hotspots: 0
💳 Added Tech Debt:  0
```

Static analysis runs on every commit. If quality drops below threshold, **the pipeline does not proceed.**

</td>
<td width="50%" valign="top">

### Trivy — Image Scan

```
✅ Status:           CLEAN
🛡️ Critical CVEs:    0
⚠️  High CVEs:        0
📦 Scanned Before:    Push to registry
```

No image reaches the container registry — let alone production — without clearing this scan.

</td>
</tr>
</table>

> **Shift-left in practice, not just in theory.** The vulnerability scan sits *before* the push stage, not after. A vulnerable image is architecturally incapable of reaching the registry.

<br/>

## 🖥️ Infrastructure

<div align="center">

| Node | Role | Type | Status |
|------|------|------|--------|
| 🔧 Jenkins | CI/CD Orchestrator | `m7i-flex.large` | 🟢 Running |
| 🧠 Master | Swarm Manager | `m7i-flex.large` | 🟢 Running |
| ⚙️ Worker 1 | Swarm Worker Node | `t3.micro` | 🟢 Running |
| ⚙️ Worker 2 | Swarm Worker Node | `t3.micro` | 🟢 Running |

</div>

All four nodes run on **AWS EC2**, orchestrated as a self-managed **Docker Swarm cluster** — one manager, two workers — with Docker Compose stack files defining service placement and Swarm handling distribution across the cluster.

<br/>

## 🧰 Tech Stack

<div align="center">

![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-4E9BCD?style=for-the-badge&logo=sonarqube&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-1904DA?style=for-the-badge&logo=aquasecurity&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Docker Swarm](https://img.shields.io/badge/Docker%20Swarm-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS%20EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)

</div>

<br/>

## 🚀 Quick Start

```bash
# Clone the repository
git clone https://github.com/jyothisai0336/Project_Zomato.git
cd Project_Zomato

# Build the image locally
docker build -t zomato:local .

# Run standalone
docker run -d -p 3000:3000 zomato:local

# OR deploy as a Swarm stack (requires an initialized swarm)
docker stack deploy -c docker-compose.yml zomato-stack
```

<details>
<summary><b>🔧 Prerequisites</b></summary>

- Docker Engine (Swarm mode enabled for cluster deployment)
- Jenkins with SonarQube Scanner + Trivy installed on agents
- A reachable SonarQube server for the CQA stage
- AWS EC2 instances (or any Docker-capable hosts) for the Swarm nodes

</details>

<br/>

## 📌 Known Limitations

- This is a **reference/demo deployment** — infrastructure is provisioned for demonstration and is not kept running permanently.
- No TLS termination is configured on the demo instances; a production deployment would sit behind a load balancer or reverse proxy with HTTPS.

<br/>

## 📬 Contact

<div align="center">

**Jyothisai Mekala**
DevOps / DevSecOps Engineer

[![Email](https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:mekalajyothisai3@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](www.linkedin.com/in/jyothisai-mekala-6852a822a)

<br/>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:FF4B2B,100:FF416C&height=100&section=footer" width="100%"/>

</div>
