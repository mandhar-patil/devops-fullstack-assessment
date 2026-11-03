## Assumptions and Design Decisions

1. The application is a demonstration Hello World application with a React frontend and Django backend.
2. Docker Compose is used to run the frontend and backend services together.
3. Nginx serves the React frontend and acts as a reverse proxy for backend API requests.
4. The frontend communicates with Django through the relative API endpoint `/api/hello/`.
5. The backend listens on port 8000 internally and is not directly exposed to the public internet.
6. The frontend is accessible through port 80 on the EC2 instance.
7. Docker Hub is used as the container image registry.
8. AWS EC2 is used as the deployment environment.
9. Environment-specific credentials and secrets are provided through environment variables and are not committed to GitHub.
10. Terraform infrastructure provisioning is considered optional unless Terraform configuration files are included in the repository.


## Documentation

See [DEVOPS.md](DEVOPS.md) for local setup, EC2 deployment, CI/CD workflow, and troubleshooting instructions.
