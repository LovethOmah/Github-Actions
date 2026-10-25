 
# End to End CI/CD Pipeline with GitHub Actions, SonarQube, Docker & Amazon EKS

A GitHub Actions pipeline that builds, scans, tests and deploys a Java Spring Boot app to Amazon EKS on every push to `main`. All jobs run on a self-hosted runner on AWS EC2.

## Pipeline flow

```
compile → security-check → test → build_project_sonar_scan → build_docker_image_and_push → deploy_to_kubernetes
```

| Job | What it does |
|---|---|
| compile | Compiles the app with Maven (JDK 17) |
| security-check | Trivy scans the code and dependencies for vulnerabilities; Gitleaks scans for leaked secrets |
| test | Runs the unit tests |
| build_project_sonar_scan | Packages the JAR, uploads it as an artifact, and runs the SonarQube scan and Quality Gate |
| build_docker_image_and_push | Downloads the JAR, builds the Docker image and pushes it to Docker Hub |
| deploy_to_kubernetes | Connects to EKS and deploys the app and MySQL with `kubectl apply -f ds.yml` |

## Infrastructure

You launch **three** EC2 servers yourself:

| Server | Purpose | Instance type |
|---|---|---|
| Runner | Runs all pipeline jobs | t3.small, 20 GB disk |
| SonarQube | Code quality analysis | m7i-flex.large (8 GB RAM) |
| EKS admin | Runs Terraform and kubectl to create and manage the cluster | t3.small |

Terraform then creates the EKS cluster and **2 worker nodes** (t3.small) automatically. The worker nodes appear in your EC2 instance list, but you **don't** create them yourself.

All servers run Ubuntu in `us-east-1`.

## Steps

### 1. Runner server

Install the tools the pipeline needs. A self-hosted runner starts with none of them:

```bash
sudo apt-get update
sudo apt-get install -y maven unzip docker.io
sudo usermod -aG docker ubuntu
```

- **maven**: builds and tests the app
- **unzip**: needed by the SonarQube scan and AWS CLI install steps
- **docker.io**: builds the Docker image; the `usermod` line lets the runner use Docker without `sudo`

Log out and SSH in again so the Docker permission applies.

Register the runner. On GitHub, go to **Settings → Actions → Runners → New self-hosted runner**, choose **Linux / x64**, and run the download and `config.sh` commands shown. Then start it:

```bash
cd ~/actions-runner
./run.sh
```

The pipeline installs **Trivy**, **Gitleaks**, **JDK 17**, the **AWS CLI** and **kubectl** itself on each run, so you don't need to install them here.

### 2. SonarQube server

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo sysctl -w vm.max_map_count=524288
sudo docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

- SonarQube needs **at least 4 GB RAM**. On a 1 GB instance it crashes.
- `vm.max_map_count` is required by Elasticsearch, which runs inside SonarQube.
- Open **port 9000** in the security group.

Open `http://<sonarqube-ip>:9000`, log in with `admin` / `admin`, set a new password, and generate a token under **My Account → Security**.

### 3. EKS admin server

Install Terraform, the AWS CLI and kubectl:

```bash
# Terraform
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y terraform

# AWS CLI
sudo apt-get install -y unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Connect the AWS CLI to your account:

```bash
aws configure
```

Create the cluster with the Terraform code in [EKS-Terraform](https://github.com/LovethOmah/EKS-Configuration-Terraform
):

```bash
git clone https://github.com/LovethOmah/EKS-Terraform.git
cd EKS-Terraform
terraform init
terraform apply
```

Then connect kubectl and check the nodes:

```bash
aws eks update-kubeconfig --region us-east-1 --name opswizard-cluster
kubectl get nodes
```

Terraform creates the cluster and its 2 worker nodes for you. They appear as extra instances in EC2; this is expected. `kubectl get nodes` should show both as `Ready`.

Copy the contents of `~/.kube/config`. You'll add it as a GitHub secret in the next step.

### 4. GitHub secrets and variables

Add these under **Settings → Secrets and variables → Actions**:

| Name | Type | Value |
|---|---|---|
| `SONAR_TOKEN` | Secret | SonarQube token |
| `SONAR_HOST_URL` | Variable | `http://<sonarqube-private-ip>:9000` |
| `DOCKERHUB_USERNAME` | Variable | Docker Hub username |
| `DOCKERHUB_TOKEN` | Secret | Docker Hub access token |
| `AWS_ACCESS_KEY_ID` | Secret | Access key of the IAM user that created the cluster |
| `AWS_SECRET_ACCESS_KEY` | Secret | Matching secret key |
| `EKS_KUBECONFIG` | Secret | Contents of `~/.kube/config` from the EKS admin server |

### 5. Run the pipeline

Push to `main`. When all six jobs pass, get the app's URL from the EKS admin server:

```bash
kubectl get pods
kubectl get svc fin-app-service
```

Open the `EXTERNAL-IP` address in a browser.

## Clean up

Run these from the EKS admin server:

```bash
kubectl delete -f ds.yml
terraform destroy
```

Delete the app before running `terraform destroy`. The load balancer Kubernetes created would otherwise block the VPC from being deleted. Then stop or terminate the runner, SonarQube and EKS admin servers.

---

The pipeline was built one stage at a time. See the [commit history](https://github.com/LovethOmah/Github-Actions_CICD_EKS/commits/main) for how each job was added.

## Screenshots

<img width="1463" height="773" alt="Screenshot 2026-10-25 at 15 22 25" src="https://github.com/user-attachments/assets/3725be08-d55b-4de0-aaad-67fcac8cf849" />



<img width="1453" height="724" alt="Screenshot 2026-10-25 at 15 10 25" src="https://github.com/user-attachments/assets/431f41d4-f6ff-4dfd-87a5-264ba84d4721" />



<img width="1472" height="474" alt="Screenshot 2026-10-25 at 15 03 12" src="https://github.com/user-attachments/assets/34d17526-7368-4466-8c0a-6e15913833b8" />



<img width="1461" height="487" alt="Screenshot 2026-10-25 at 15 02 05" src="https://github.com/user-attachments/assets/16bb5d59-6761-449e-99a3-9f5ca309402a" />


<img width="1395" height="756" alt="Screenshot 2026-10-25 at 14 54 01" src="https://github.com/user-attachments/assets/46bfbb31-bb5a-431e-baf7-ea9dd0a8e27e" />



<img width="1351" height="827" alt="Screenshot 2026-10-25 at 14 49 53" src="https://github.com/user-attachments/assets/a731806a-eb55-4683-b86f-6ee1042cbcbf" />






