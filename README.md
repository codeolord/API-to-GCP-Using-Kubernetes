# API-to-GCP-Using-Kubernetes
Developing the API:

We can create a simple API using Python and Flask. First, create a directory for your project and create a file named app.py.

# app.py  
from flask import Flask, jsonify  
from datetime import datetime  

app = Flask(__name__)  

@app.route('/time', methods=['GET'])  
def get_current_time():  
    current_time = datetime.utcnow().isoformat() + "Z"  # Return current time in UTC  
    return jsonify({"current_time": current_time})  

if __name__ == '__main__':  
    app.run(host='0.0.0.0', port=8080)  
Create a Dockerfile in the same directory to containerize the API:

# Dockerfile  
FROM python:3.9-slim  

WORKDIR /app  
COPY requirements.txt requirements.txt  
RUN pip install --no-cache-dir -r requirements.txt  
COPY . .  

EXPOSE 8080  

CMD ["python", "app.py"]  
Create a requirements.txt file to specify the Flask dependency:

Flask==2.2.2  
Test the API Locally:

You can build and test the API locally before deploying:

docker build -t time-api .  
docker run -p 8080:8080 time-api  
Access the API at http://localhost:8080/time.

Step 2: Terraform for Infrastructure Deployment
Setup Terraform:

Create a main.tf file in your project directory for Terraform configurations:

terraform {  
    required_providers {  
        google = {  
            source  = "hashicorp/google"  
            version = "~> 3.5"  
        }  
        kubernetes = {  
            source  = "hashicorp/kubernetes"  
            version = "~> 2.0"  
        }  
    }  
    required_version = ">= 0.12"  
}  

provider "google" {  
    credentials = file("<path_to_your_service_account_key>.json")  
    project     = "<your_project_id>"  
    region      = "<your_region>"  
}  

resource "google_compute_network" "vpc_network" {  
    name = "my-vpc-network"  
    auto_create_subnetworks = false  
}  

resource "google_compute_subnetwork" "subnetwork" {  
    name          = "my-subnetwork"  
    ip_cidr_range = "10.0.0.0/24"  
    region        = "<your_region>"  
    network       = google_compute_network.vpc_network.name  
}  

resource "google_service_networking_connection" "private_service_access" {  
    service = "servicenetworking.googleapis.com"  
    reserved_ip_range = google_compute_subnetwork.subnetwork.ip_cidr_range  
    network = google_compute_network.vpc_network.name  
}  

resource "google_container_cluster" "primary" {  
    name     = "my-cluster"  
    location = "<your_region>"  

    initial_node_count = 1  

    master_auth {  
        username = "admin"  
        password = "password"  # Change this  
    }  

    remove_default_node_pool = true  
    gateway_ipv4_cidr_block = "10.64.0.0/28"  
    
    node_pool {  
        name = "default-pool"  
        initial_node_count = 1  
        management {  
            auto_upgrade = true  
            auto_repair  = true  
        }  

        node_config {  
            machine_type = "e2-medium"  
            oauth_scopes = [  
                "https://www.googleapis.com/auth/cloud-platform",  
            ]  
        }  
    }  
}  

resource "google_compute_router" "nat_router" {  
    name    = "nat-router"  
    region  = "<your_region>"  
    network = google_compute_network.vpc_network.name  
}  

resource "google_compute_router_nat" "nat_gateway" {  
    name                              = "nat-gateway"  
    router                            = google_compute_router.nat_router.name  
    region                            = google_compute_router.nat_router.region  
    nat_ip_allocate_option            = "MANUAL_ONLY"  
    nat_ips                           = ["<your_external_ip>"]  

    log_config {  
        enabled = true  
        filter  = "ERRORS_ONLY" # or ALL  
    }  
}  

resource "kubernetes_namespace" "api_namespace" {  
    metadata {  
        name = "api-namespace"  
    }  
}  

resource "kubernetes_deployment" "time_api" {  
    metadata {  
        name      = "time-api"  
        namespace = kubernetes_namespace.api_namespace.metadata[0].name  
    }  
    spec {  
        replicas = 2  
        selector {  
            match_labels = {  
                app = "time-api"  
            }  
        }  
        template {  
            metadata {  
                labels = {  
                    app = "time-api"  
                }  
            }  
            spec {  
                container {  
                    name  = "time-api"  
                    image = "<gcr.io>/<your_project_id>/time-api:latest"  
                    port {  
                        container_port = 8080  
                    }  
                }  
            }  
        }  
    }  
}  

resource "kubernetes_service" "time_api_service" {  
    metadata {  
        name      = "time-api-service"  
        namespace = kubernetes_namespace.api_namespace.metadata[0].name  
    }  
    spec {  
        type = "LoadBalancer"  
        selector = {  
            app = kubernetes_deployment.time_api.metadata[0].labels["app"]  
        }  
        port {  
            port        = 80  
            target_port = 8080  
        }  
    }  
}  

resource "kubernetes_ingress" "time_api_ingress" {  
    metadata {  
        name      = "time-api-ingress"  
        namespace = kubernetes_namespace.api_namespace.metadata[0].name  
    }  
    spec {  
        rule {  
            host = "<your_domain_or_ip>"  
            http {  
                path {  
                    path = "/time"  
                    path_type = "Prefix"  
                    backend {  
                        service {  
                            name = kubernetes_service.time_api_service.metadata[0].name  
                            port {  
                                number = 80  
                            }  
                        }  
                    }  
                }  
            }  
        }  
    }  
}  

resource "google_project_iam_member" "service_account_role" {  
    project = "<your_project_id>"  
    role    = "roles/container.admin"  
    member  = "serviceAccount:<your_service_account_email>"  
}  

output "time_api_url" {  
    value = kubernetes_service.time_api_service.status[0].load_balancer[0].ingress[0].ip  
}  
Ensure to replace the placeholders in the Terraform configuration with your specific values.

Initialize Terraform:

Run the following commands to initialize and apply your Terraform configurations:

terraform init  
terraform apply  
Confirm the prompts, and Terraform will create the resources in GCP.

Step 3: CI/CD with GitHub Actions
Create a directory called .github/workflows in your project root and create a YAML file called ci-cd.yml in it:

name: CI/CD Pipeline  

on:  
  push:  
    branches:  
      - main  

jobs:  
  build:  
    runs-on: ubuntu-latest  

    steps:  
    - name: Checkout code  
      uses: actions/checkout@v2  

    - name: Set up Docker Buildx  
      uses: docker/setup-buildx-action@v1  

    - name: Log in to the Docker registry  
      uses: docker/login-action@v1  
      with:  
        username: _json_key  
        password: ${{ secrets.GCP_SA_KEY }}  

    - name: Build Docker image  
      run: |   
        docker build -t gcr.io/<your_project_id>/time-api:latest .  
        docker push gcr.io/<your_project_id>/time-api:latest  

    - name: Terraform Setup  
      uses: hashicorp/setup-terraform@v1  
      with:  
        terraform_version: 1.0.0  

    - name: Terraform Init  
      run: terraform init  

    - name: Terraform Apply  
      run: terraform apply -auto-approve  
      env:  
        GOOGLE_APPLICATION_CREDENTIALS: ${{ secrets.GCP_SA_KEY }}  
Step 4: Network Security
Your Terraform configuration must contain a NAT gateway and firewall rules. A simple firewall rule could look like this:

resource "google_compute_firewall" "allow_http" {  
   name    = "allow-http"  
   network = google_compute_network.vpc_network.name  

   allow {  
      protocol = "tcp"  
      ports    = ["80", "443"]  
   }  

   source_ranges = ["0.0.0.0/0"]  
}  
Step 5: Deliverables:
GitHub Repository Setup:

Commit all your code and create a new repository on GitHub.
Include:
Terraform code (main.tf)
Dockerfile
requirements.txt
.github/workflows/ci-cd.yml
README.md with instructions on local testing and setup.
Deployment and Testing:

Follow the README instructions for local testing and verify the functionality.
Push to GitHub and monitor your actions running workflow and deploying your infrastructure.
Collect Links:

The final deliverables should include:
The link to your GitHub repository.
The deployed API endpoint.
The link to the GitHub Actions workflow run.
