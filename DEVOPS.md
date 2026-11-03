# DEVOPS.md

# Full-Stack Hello World Application – DevOps Documentation

## 1. Project Overview

This project demonstrates the containerization, automation, and deployment of a full-stack Hello World application using Django (Backend) and React (Frontend).

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

### Application Architecture

1. The user accesses the application through the EC2 public IP.
2. Nginx serves the React frontend.
3. Nginx forwards `/api/` requests to the Django backend.
4. Django processes the request and returns a JSON response.
5. Docker Compose manages the frontend and backend containers.

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

Verify installations:

```bash
git --version
docker --version
docker compose version
node --version
npm --version
python3 --version
```

## 3. Project Setup – Local Environment

### Step 1: Clone the repository

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git

cd YOUR_REPOSITORY
```

### Step 2: Configure environment variables

Create a `.env` file using the project's environment variable template.

Example:

```env
DOCKERHUB_USERNAME=your_dockerhub_username
DJANGO_SECRET_KEY=replace_with_a_secure_secret
```

Never commit `.env` files or production secrets to GitHub.

### Step 3: Build and start the application

```bash
docker compose up -d --build
```

### Step 4: Verify running containers

```bash
docker compose ps
```

### Step 5: Check application logs

```bash
docker compose logs -f
```

### Step 6: Test the backend API

```bash
curl http://localhost/api/hello/
```

Expected response:

```json
{
  "message": "Hello World from Django Backend!"
}
```

### Step 7: Access the application

Open:

```text
http://localhost
```

## 4. Deployment – AWS EC2

### Step 1: Launch an EC2 instance

- Launch an Ubuntu EC2 instance.
- Configure SSH access.
- Allow inbound HTTP traffic on port 80.
- Keep SSH access restricted to your IP address.

### Step 2: Connect to EC2

```bash
ssh -i your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

### Step 3: Install Docker

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

### Step 4: Clone the project

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY.git

cd YOUR_REPOSITORY
```

### Step 5: Configure environment variables

Create the production `.env` file with the required secure values.

Ensure that Django `ALLOWED_HOSTS` includes the required EC2 hostname/IP and the internal Docker service hostname `backend`.

### Step 6: Start the application

```bash
docker compose up -d --build
```

### Step 7: Verify deployment

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

### Step 8: Access the deployed application

```text
http://YOUR_EC2_PUBLIC_IP
```

## 5. CI/CD Pipeline

GitHub Actions is used to automate the application build and deployment workflow.

The intended pipeline workflow includes:

1. Developer pushes code to GitHub.
2. GitHub Actions detects the push.
3. The workflow checks out the source code.
4. Docker images are built for the frontend and backend.
5. Images are pushed to Docker Hub using GitHub Secrets.
6. The deployment stage updates the application on the server, if configured.

### Required GitHub Secrets

Configure the required credentials under:

GitHub Repository → Settings → Secrets and variables → Actions.

Examples:

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
- `EC2_HOST`
- `EC2_USERNAME`
- `EC2_SSH_KEY`

Only add secrets that are actually referenced by your workflow.

Never hardcode passwords, private keys, or access tokens in workflow files.

### Verify CI/CD execution

1. Open the GitHub repository.
2. Select the Actions tab.
3. Open the latest workflow execution.
4. Verify that all configured jobs complete successfully.
5. Push a fresh commit and verify that the workflow triggers again.

## 6. Troubleshooting Log

During deployment, I encountered a frontend-to-backend connectivity issue.

### Issue 1: Frontend displayed Connection Failed

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

**Root cause:**

The React application was using:

```typescript
axios.get('http://localhost:8000/api/hello/')
```

The browser interpreted `localhost` as the user's own computer rather than the EC2 backend.

**Solution:**

Updated `frontend/src/App.tsx`:

```typescript
axios.get('/api/hello/')
```

This allowed the browser to send the API request to Nginx on the same origin.

### Issue 2: Django returned HTTP 400 Bad Request

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

**Root cause:**

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

## 7. Security Best Practices

- Run application containers as non-root users.
- Use multi-stage Docker builds where appropriate.
- Maintain `.dockerignore` files.
- Never commit `.env` files or secrets.
- Use GitHub Actions Secrets for CI/CD credentials.
- Restrict EC2 security group access.
- Use specific Django `ALLOWED_HOSTS` values.
- Use versioned Docker image tags.
- Avoid exposing the Django backend port publicly unless required.

## 8. Useful Docker Commands

Check running containers:

```bash
docker ps
```

Check all containers:

```bash
docker ps -a
```

View container logs:

```bash
docker logs CONTAINER_NAME
```

Stop application:

```bash
docker compose down
```

Restart application:

```bash
docker compose restart
```

Rebuild application:

```bash
docker compose up -d --build
```

Check Docker images:

```bash
docker images
```

Check Docker networks:

```bash
docker network ls
```

## 9. Deployment Information

**GitHub Repository:**  
https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY

**Deployed Application:**  
http://YOUR_EC2_PUBLIC_IP

**Application API:**  
http://YOUR_EC2_PUBLIC_IP/api/hello/

## 10. Conclusion

This project demonstrates the implementation of a full-stack application deployment using Docker, Docker Compose, Nginx, Django, React, AWS EC2, and GitHub Actions.

The deployment process included containerization, backend/frontend communication, reverse proxy configuration, CI/CD automation, and troubleshooting.

The project helped demonstrate practical DevOps skills in application deployment, container management, infrastructure operations, and production troubleshooting.
