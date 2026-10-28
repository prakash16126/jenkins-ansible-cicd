# Jenkins Ansible CI/CD

A sample Java web application packaged as a WAR and deployed through a CI/CD pipeline built with Jenkins, Maven, Docker, and Ansible. This repository demonstrates an end-to-end DevOps workflow where a Java application is built, stored, and deployed to a Tomcat server with automated health checks.

## Overview

This project combines the following technologies:

- Java 17
- Maven
- Apache Tomcat 10.1
- Jenkins pipeline
- Ansible
- Docker
- Linux-based deployment environment (Ubuntu/Debian style)

The application is built as a web project, and its artifact is named using Maven metadata from `pom.xml`:

- Group ID: `in.RAHAM`
- Artifact ID: `NETFLIX`
- Version: `1.2.2`
- Packaging: `war`

This produces a WAR artifact named:

```text
NETFLIX-1.2.2.war
```

The application is intended to be deployed to Tomcat under the `NETFLIX` context.

## Repository Structure

```text
jenkins-ansible-cicd/
├── Dockerfile
├── Jenkinsfile
├── pom.xml
├── .gitignore
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini.example
│   └── playbooks/
│       ├── deploy-app.yml
│       ├── health-check.yml
│       └── install-tomcat.yml
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── in/
│   │   │       └── javahome/
│   │   │           └── myweb/
│   │   │               └── controller/
│   │   └── webapp/
│   └── test/
│       └── java/
└── target/   (generated after Maven build)
```

## Prerequisites

Before working with this project, ensure the following are installed:

- Java 17
- Maven
- Docker
- Jenkins (for CI/CD execution)
- Ansible
- AWS CLI (used in the Jenkins pipeline for S3 upload)
- Access to a target Linux host for Tomcat deployment

## Build the Project Locally

From the repository root, run:

```bash
mvn clean package -DskipTests
```

This generates the WAR artifact in the `target` directory.

To compile without packaging:

```bash
mvn compile
```

To run tests:

```bash
mvn test
```

## Run with Docker

This repository includes a multi-stage `Dockerfile`:

- Stage 1: Builds the WAR using Maven
- Stage 2: Deploys the WAR to a Tomcat runtime container

Build the image:

```bash
docker build -t netflix-app .
```

Run the container:

```bash
docker run -d -p 8080:8080 --name netflix-app netflix-app
```

Then access the application at:

```text
http://localhost:8080/NETFLIX
```

The Docker image exposes port `8080` and starts Tomcat using:

```bash
catalina.sh run
```

## Jenkins Pipeline

The `Jenkinsfile` defines a pipeline with the following stages:

1. `clean`
   - Clears the workspace
2. `checkout`
   - Checks out the application source
3. `build`
   - Runs `mvn compile`
4. `test`
   - Runs `mvn test`
5. `artifact`
   - Runs `mvn package`
6. `s3`
   - Uploads the generated WAR artifact to an S3 bucket
7. `deploy`
   - Deploys the WAR to Tomcat using the Jenkins Tomcat deploy plugin

### Jenkins Credentials

The pipeline expects credentials such as:

- `AWS_ACCEESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `tomcat`

These should be configured in Jenkins before running the pipeline.

### Notes

The Jenkins pipeline currently references a WAR path similar to:

```text
/var/lib/jenkins/workspace/ci-pipeline/target/NETFLIX-1.2.2.war
```

It also uploads the artifact to an S3 bucket such as:

```text
s3://k8s-cluster-backup-008
```

The deployment target is configured to a Tomcat URL such as:

```text
http://3.7.70.46:8080/
```

These values may need to be updated to match your environment.

## Ansible Automation

The `ansible/` directory contains deployment automation for installing and configuring Tomcat and deploying the WAR file.

### Directory Contents

```text
ansible/
├── ansible.cfg
├── inventory.ini.example
└── playbooks/
    ├── deploy-app.yml
    ├── health-check.yml
    └── install-tomcat.yml
```

### 1) Install Tomcat

The playbook `ansible/playbooks/install-tomcat.yml` performs the following actions:

- Installs Java 17
- Installs required OS packages (`wget`, `tar`)
- Creates a `tomcat` system user and group
- Downloads Apache Tomcat 10.1.44
- Extracts it into `/opt/tomcat`
- Creates a systemd service for Tomcat
- Starts and enables Tomcat

Run it with:

```bash
cp ansible/inventory.ini.example ansible/inventory.ini
```

Then update the inventory file for your target host(s) and run:

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbooks/install-tomcat.yml
```

### 2) Deploy the Application

The playbook `ansible/playbooks/deploy-app.yml`:

- Checks whether the WAR exists on the Jenkins server
- Stops Tomcat
- Removes the previous app directory and WAR
- Copies the new WAR into the Tomcat `webapps` directory
- Starts Tomcat
- Waits for the app port to open
- Verifies the service is active

Run it with:

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbooks/deploy-app.yml
```

### 3) Health Check

The playbook `ansible/playbooks/health-check.yml` validates the application by checking:

```text
http://localhost:8080/NETFLIX/
```

and ensures it responds with HTTP status `200`.

Run it with:

```bash
ansible-playbook -i ansible/inventory.ini ansible/playbooks/health-check.yml
```

## Deployment Flow

A typical CI/CD flow for this repository is:

```text
Developer changes code
       |
       v
Jenkins pipeline starts
       |
       +--> Maven compile
       +--> Maven test
       +--> Maven package
       +--> Build WAR artifact
       +--> Upload to S3
       +--> Deploy to Tomcat
       +--> Run Ansible deployment
       +--> Perform health check
```

## Configuration Notes

### Example Inventory

Use `ansible/inventory.ini.example` as a template and create your own inventory file:

```ini
[dev]
your-host-or-ip ansible_user=ubuntu ansible_become=true
```

### Tomcat Directory

The deployment playbook uses:

```text
/opt/tomcat
```

for the Tomcat installation and deployment target.

### Application URL

The health-check playbook checks:

```text
http://localhost:8080/NETFLIX/
```

This file is deployed as a WAR to Tomcat and should be served under the `NETFLIX` context.

## Troubleshooting

### Maven Build Fails

Make sure:

- Java 17 is installed
- Maven is installed and available in `PATH`
- The environment has internet access to download dependencies

Check with:

```bash
java -version
mvn -version
```

### Docker Build Fails

Ensure Docker is running and that your system has enough resources to build the image.

### Tomcat is Not Starting

Verify service status:

```bash
systemctl status tomcat
```

Check logs:

```bash
journalctl -u tomcat -n 100 --no-pager
```

### Application Not Reachable

Confirm:

- Tomcat is running
- The WAR file was successfully copied to `/opt/tomcat/webapps`
- The file is named `NETFLIX.war`
- The application path is correct: `http://host:8080/NETFLIX/`

### Ansible Fails

Check:

```bash
ansible --version
ansible-playbook --syntax-check ansible/playbooks/deploy-app.yml
```

Verify SSH connectivity to the target host and validate the inventory file.

## Security Considerations

This repository includes example credential usage in Jenkins, which is appropriate for learning but should be handled carefully in production:

- Store secrets in Jenkins credentials or a secret manager
- Avoid committing real credentials to source control
- Restrict access to deployment hosts and AWS resources
- Use least-privilege IAM policies for AWS access

## Use Cases

This repository is useful for learning and demonstrating:

- Java web application CI/CD automation
- Jenkins pipeline design
- Maven-based WAR packaging
- Dockerized build/deploy patterns
- Tomcat provisioning using Ansible
- Automated deployment verification

## Summary

This project is a practical example of a Java-based CI/CD pipeline using Jenkins and Ansible. It shows how to build a WAR artifact, provision Tomcat on remote hosts, deploy the application, and verify it is healthy after deployment.

It is a strong example of a DevOps-style workflow for Java web applications in a Linux environment.

## License

This repository does not include an explicit license file in the repository snapshot reviewed here. If this project is intended for public reuse, consider adding an appropriate open-source license such as MIT or Apache 2.0.
