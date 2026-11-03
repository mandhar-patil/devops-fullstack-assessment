# DEVOPS.md

# Full-Stack Hello World Application – DevOps Documentation

## 1. Project Overview

This project demonstrates the containerization, CI/CD automation, and deployment of a full-stack Hello World application using Django (Backend) and React (Frontend).

The application is containerized using Docker and Docker Compose. Jenkins is used to automate the CI/CD pipeline, while Docker Hub stores the application images.

### Technology Stack

| Component | Technology |
|---|---|
| Frontend | React, TypeScript |
| Backend | Python, Django |
| Application Server | Gunicorn |
| Web Server / Reverse Proxy | Nginx |
| Containerization | Docker |
| Container Orchestration | Docker Compose |
| CI/CD | Jenkins |
| Cloud Infrastructure | AWS EC2 |
| Container Registry | Docker Hub |
| Source Code Management | GitHub |

### Application Architecture

1. The user accesses the application through the EC2 public IP.
2. Nginx serves the React frontend.
3. Nginx forwards `/api/` requests to the Django backend.
4. Django processes the request and returns a JSON response.
5. Docker Compose manages the frontend and backend containers.
6. Jenkins automates building, pushing, and deploying Docker images.

---

## 2. Prerequisites

Install the following tools:

- Git
- Docker
- Docker Compose plugin
- Node.js and npm (for local frontend development)
- Python 3 and pip (for local backend development)
- AWS EC2 account
- GitHub account
- Docker Hub account
- Jenkins

### Verify Installations

```bash
git --version
docker --version
docker compose version
node --version
npm --version
python3 --version
```

---

## 3. Project Setup – Local Environment

### Step 1: Clone the Repository

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git

cd YOUR_REPOSITORY
```

### Step 2: Configure Environment Variables

Create a `.env` file using the project's environment variable template.

Example:

```env
DOCKERHUB_USER=mandhar12
DJANGO_SECRET_KEY=replace_with_a_secure_secret
DEBUG=False
ALLOWED_HOSTS=localhost,127.0.0.1,backend
API_URL=/api
```

Never commit `.env` files or production secrets to GitHub.

### Step 3: Build and Start the Application

```bash
docker compose up -d --build
```

### Step 4: Verify Running Containers

```bash
docker compose ps
```

### Step 5: Check Application Logs

```bash
docker compose logs -f
```

### Step 6: Test the Backend API

```bash
curl http://localhost/api/hello/
```

Expected response:

```json
{
  "message": "Hello World from Django Backend!"
}
```

### Step 7: Access the Application

Open:

```text
http://localhost
```

---

## 4. Deployment – AWS EC2

### Step 1: Launch an EC2 Instance

Launch an Ubuntu EC2 instance.

Configure the following:

- SSH access on port 22, restricted to your IP.
- HTTP access on port 80.
- Appropriate instance type and storage.

### Step 2: Connect to EC2

```bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

### Step 3: Install Docker and Git

```bash
sudo apt update

sudo apt install -y docker.io docker-compose-v2 git

sudo systemctl enable --now docker

sudo usermod -aG docker ubuntu
```

Reconnect to the EC2 instance after adding the user to the Docker group.

Verify:

```bash
docker --version

docker compose version
```

### Step 4: Clone the Project

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git

cd YOUR_REPOSITORY
```

### Step 5: Configure Environment Variables

Configure the required production environment variables.

Ensure that:

- `DJANGO_SECRET_KEY` contains a secure secret.
- `DEBUG=False` is configured.
- Django `ALLOWED_HOSTS` includes the required EC2 hostname/IP and internal Docker service hostname.
- `API_URL=/api` is configured where required by the frontend.

### Step 6: Start the Application

```bash
docker compose up -d --build
```

### Step 7: Verify Deployment

```bash
docker compose ps

docker compose logs --tail=50
```

Test the API:

```bash
curl -i http://localhost/api/hello/
```

Expected HTTP status:

```text
HTTP/1.1 200 OK
```

### Step 8: Access the Deployed Application

```text
http://YOUR_EC2_PUBLIC_IP
```

---

## 5. CI/CD Pipeline – Jenkins

Jenkins is used to automate the complete CI/CD process.

The pipeline is implemented using a declarative Jenkinsfile.

### Jenkins CI/CD Workflow

The pipeline follows these stages:

1. Developer pushes code to the GitHub repository.
2. Jenkins detects repository changes using SCM polling.
3. Jenkins checks out the latest source code.
4. Jenkins builds the backend Docker image.
5. Jenkins builds the frontend Docker image.
6. Jenkins authenticates with Docker Hub using Jenkins credentials.
7. Jenkins pushes both images to Docker Hub.
8. Jenkins deploys the latest images using Docker Compose.
9. Jenkins removes unused dangling Docker images.

### Pipeline Architecture

```text
       Developer
           |
           v
     GitHub Repository
           |
           v
    Jenkins SCM Polling
      (Every 2 Minutes)
           |
           v
       Checkout
           |
           v
      Build Images
           |
       +---+---+
       |       |
       v       v
    Backend  Frontend
     Image     Image
       |       |
       +---+---+
           |
           v
      Docker Hub
           |
           v
        Deploy
           |
           v
     Docker Compose
           |
           v
    Running Application
        on EC2
```

### Jenkins Pipeline Stages

#### Stage 1: Checkout

Jenkins checks out the latest source code from the configured GitHub repository.

```groovy
stage('Checkout') {
    steps {
        checkout scm
    }
}
```

#### Stage 2: Build Images

Jenkins builds Docker images for the frontend and backend.

```groovy
docker build -t mandhar12/devops-backend:$IMAGE_TAG ./backend

docker build -t mandhar12/devops-frontend:$IMAGE_TAG ./frontend
```

Each image receives two tags:

- Build number tag.
- Latest tag.

Example:

```text
mandhar12/devops-backend:15
mandhar12/devops-backend:latest

mandhar12/devops-frontend:15
mandhar12/devops-frontend:latest
```

#### Stage 3: Push Images

Jenkins authenticates with Docker Hub using stored Jenkins credentials.

Credential ID:

```text
dockerhub-creds
```

Jenkins pushes the versioned and latest images to Docker Hub.

#### Stage 4: Deploy

Jenkins securely injects the Django secret key using Jenkins credentials.

The pipeline executes:

```bash
docker compose pull

docker compose up -d --remove-orphans

docker image prune -f
```

This pulls the latest images from Docker Hub and updates the running application using Docker Compose.

### Jenkins Credentials Configuration

Configure the following credentials in Jenkins:

Navigate to:

```text
Jenkins Dashboard
    → Manage Jenkins
    → Credentials
    → System
    → Global credentials
    → Add Credentials
```

**Credential 1: Docker Hub**

| Setting | Value |
|---|---|
| Kind | Username with password |
| ID | dockerhub-creds |
| Username | Your Docker Hub username |
| Password | Docker Hub access token |

**Credential 2: Django Secret Key**

| Setting | Value |
|---|---|
| Kind | Secret text |
| ID | django-secret-key |
| Secret | Your Django secret key |

Never hardcode passwords, private keys, or access tokens in Jenkinsfile.

### SCM Polling Configuration

The Jenkinsfile uses:

```groovy
triggers {
    pollSCM('H/2 * * * *')
}
```

This checks the GitHub repository approximately every two minutes for changes.

**Why SCM Polling instead of Webhooks?**

Port 8080 on the Jenkins server is closed to public internet access.

Therefore, GitHub cannot directly send webhook requests to Jenkins.

SCM polling allows Jenkins to periodically check the repository for new commits without requiring an externally accessible webhook endpoint.

### Pipeline Execution

To execute the pipeline:

1. Open Jenkins Dashboard.
2. Select the configured pipeline job.
3. Click Build Now.
4. Open the latest build.
5. Select Console Output.
6. Verify that all stages complete successfully.

### Verify Docker Hub Images

Open Docker Hub and check the following repositories:

```text
mandhar12/devops-backend

mandhar12/devops-frontend
```

Verify that both `latest` and build-number tags are available.

---

## 6. Troubleshooting Log

During deployment, I encountered frontend-to-backend connectivity and Django host configuration issues.

### Issue 1: Frontend Displayed Connection Failed

**Problem:**

The application displayed:

```text
Connection Failed

Failed to connect to the backend.
Please ensure the Django server is running.
```

**Investigation:**

Checked running containers:

```bash
docker ps
```

Both frontend and backend containers were running.

Checked Django logs:

```bash
docker logs fullstack-pipeline-backend-1
```

Gunicorn was successfully listening on:

```text
0.0.0.0:8000
```

Tested the API internally:

```bash
python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:8000/api/hello/').read().decode())"
```

The Django API returned a successful JSON response.

**Root Cause:**

The React application was using:

```javascript
axios.get('http://localhost:8000/api/hello/')
```

The browser interpreted localhost as the user's own computer rather than the EC2 backend.

**Solution:**

Updated:

```text
frontend/src/App.tsx
```

Changed the API request to:

```javascript
axios.get('/api/hello/')
```

This allowed the browser to send the API request to Nginx on the same origin.

### Issue 2: Django Returned HTTP 400 Bad Request

**Problem:**

The frontend container could resolve the backend hostname but received:

```text
HTTP/1.1 400 Bad Request
```

**Investigation:**

Tested:

```bash
wget -O- http://backend:8000/api/hello/
```

Docker networking resolved the hostname successfully.

**Root Cause:**

Django was rejecting the internal Docker hostname because it was not configured in `ALLOWED_HOSTS`.

**Solution:**

Updated Django settings:

```python
ALLOWED_HOSTS = [
    'localhost',
    '127.0.0.1',
    'backend',
    'YOUR_EC2_PUBLIC_IP'
]
```

Rebuilt the backend:

```bash
docker compose up -d --build backend
```

### Final Verification

Tested the application through Nginx:

```bash
curl -i http://localhost/api/hello/
```

Successful response:

```text
HTTP/1.1 200 OK

{"message": "Hello World from Django Backend!"}
```

The issue was resolved by correcting the frontend API URL and Django host configuration.

---

## 7. Security Best Practices

- Run application containers as non-root users.
- Use multi-stage Docker builds where appropriate.
- Maintain `.dockerignore` files.
- Never commit `.env` files or secrets.
- Store Docker Hub credentials in Jenkins Credentials.
- Store Django secret keys securely in Jenkins Credentials.
- Restrict EC2 security group access.
- Use specific Django `ALLOWED_HOSTS` values instead of wildcards.
- Use versioned Docker image tags.
- Avoid exposing the Django backend port publicly unless required.
- Keep Jenkins and Docker updated.
- Use HTTPS for production application access.

**Important:** The current Jenkins deployment stage sets `ALLOWED_HOSTS='*'` for convenience. For production, replace this with specific trusted hostnames and IP addresses.

---

## 8. Useful Docker Commands

### Check Running Containers

```bash
docker ps
```

### Check All Containers

```bash
docker ps -a
```

### View Container Logs

```bash
docker logs CONTAINER_NAME
```

### View Docker Compose Logs

```bash
docker compose logs -f
```

### Stop Application

```bash
docker compose down
```

### Restart Application

```bash
docker compose restart
```

### Rebuild Application

```bash
docker compose up -d --build
```

### Pull Latest Images

```bash
docker compose pull
```

### Check Docker Images

```bash
docker images
```

### Check Docker Networks

```bash
docker network ls
```

### Remove Dangling Images

```bash
docker image prune -f
```

---

## 9. Deployment Information

**GitHub Repository:**

```text
https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY
```

**Docker Hub Backend Image:**

```text
https://hub.docker.com/r/mandhar12/devops-backend
```

**Docker Hub Frontend Image:**

```text
https://hub.docker.com/r/mandhar12/devops-frontend
```

**Deployed Application:**

```text
http://YOUR_EC2_PUBLIC_IP
```

**Application API:**

```text
http://YOUR_EC2_PUBLIC_IP/api/hello/
```

**CI/CD Tool:**

```text
Jenkins
```

---

## 10. Conclusion

This project demonstrates the implementation of an automated full-stack application deployment using Django, React, Docker, Docker Compose, Nginx, Jenkins, Docker Hub, and AWS EC2.

The project includes:

- Containerization of frontend and backend applications.
- Automated CI/CD pipeline using Jenkins.
- Automated Docker image building and tagging.
- Docker Hub image publishing.
- Automated deployment using Docker Compose.
- Jenkins credential management.
- Frontend and backend connectivity troubleshooting.
- Application deployment and verification on AWS EC2.

Through this project, I gained practical experience in CI/CD automation, Docker container management, Jenkins pipeline implementation, cloud deployment, and production troubleshooting.
