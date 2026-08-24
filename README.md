
# 🚗 End-to-End MLOps: Vehicle Insurance Prediction

[![Python 3.10](https://img.shields.io/badge/Python-3.10-blue.svg)](https://www.python.org/downloads/release/python-3100/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-green.svg)](https://fastapi.tiangolo.com/)
[![Docker](https://img.shields.io/badge/Docker-Enabled-blue.svg)](https://www.docker.com/)
[![AWS](https://img.shields.io/badge/AWS-S3%20%7C%20EC2%20%7C%20ECR-orange.svg)](https://aws.amazon.com/)

## 📖 Problem Statement
Insurance companies often possess large customer bases for health or life insurance, presenting a massive opportunity for cross-selling **Vehicle Insurance**. However, cold-calling every customer is highly inefficient and resource-intensive. 

**The Solution:** This project provides an automated, end-to-end Machine Learning pipeline that predicts whether a customer will be interested in purchasing vehicle insurance. By analyzing demographic data, vehicle age, damage history, and policy details, sales teams can prioritize high-probability leads, optimizing marketing budgets and maximizing conversion rates.

Beyond predictions, this repository serves as a **Production-Grade MLOps Architecture**, demonstrating automated data ingestion, model versioning, cloud deployment, and CI/CD automation.

---
## Video 

https://github.com/user-attachments/assets/cf21726f-6eef-40f9-8291-8611707798c1

## 🏗️ Architecture & Pipeline
This project is built using a robust, highly scalable MLOps architecture.

![pipeline](Diagrams/Pipeline.png)

### The ML Lifecycle
1. **Data Ingestion:** Automatically establishes a connection to a **MongoDB Atlas** cluster, fetching the latest raw customer data into a local feature store.
2. **Data Validation:** Enforces strict data schemas (`config/schema.yaml`) to ensure data integrity and prevent pipeline failures from drifting data types.
3. **Data Transformation:** Cleans, imputes, and scales features, saving the preprocessing object for consistent inference.
4. **Model Training & Evaluation:** Trains the predictive model and evaluates its accuracy against the current production model. 
5. **Model Registry (AWS S3):** If the newly trained model outperforms the old one, it is automatically pushed to an **AWS S3 Bucket** which acts as our central model registry.
6. **Inference & Serving:** A **FastAPI** web application loads the best model directly from AWS S3 to serve real-time predictions via a clean, interactive UI.

---

## 🛠️ Tech Stack
* **Machine Learning:** Scikit-Learn, Pandas, NumPy, Imbalanced-learn
* **Web Framework:** FastAPI, Uvicorn, Jinja2 Templates, Bootstrap 5
* **Database:** MongoDB Atlas
* **Cloud Infrastructure:** AWS S3 (Model Registry), AWS EC2 (Hosting), AWS ECR (Container Registry)
* **DevOps & MLOps:** Docker, GitHub Actions (CI/CD)

---

## 📂 Project Directory Structure
```text
.
├── .github/workflows/aws.yaml       # CI/CD pipeline configuration for AWS deployment
├── config                           # Model configurations and Data Schema validation rules
├── notebook                         # EDA, feature engineering, and MongoDB testing notebooks
├── src                              # Core pipeline source code
│   ├── cloud_storage                # AWS S3 interaction logic
│   ├── components                   # Pipeline stages (Ingestion, Transformation, Trainer, etc.)
│   ├── configuration                # AWS and MongoDB connection handlers
│   ├── constants                    # Project-wide constants and environment keys
│   ├── entity                       # Data classes for configs and artifacts
│   ├── pipline                      # End-to-end Training and Prediction pipelines
│   └── exception, logger            # Custom exception handling and logging infrastructure
├── static, templates                # CSS and HTML for the FastAPI web interface
├── app.py                           # FastAPI application entry point
├── Dockerfile                       # Containerization blueprint
└── requirements.txt                 # Python dependencies
```

---

## 🚀 Local Setup & Installation

### 1. Prerequisites
- Python 3.10
- A MongoDB Atlas Account (M0 Free Cluster is sufficient)
- AWS Account (IAM user with S3 access)

### 2. Clone and Install
```bash
git clone <your-repo-url>
cd YT-MLops-Proj1

# Create and activate a virtual environment
conda create -n vehicle python=3.10 -y
conda activate vehicle

# Install dependencies
pip install -r requirements.txt
```

### 3. Environment Variables
You must set the following environment variables to connect to the database and cloud storage:
```bash
export MONGODB_URL="mongodb+srv://<username>:<password>@cluster.mongodb.net/..."
export AWS_ACCESS_KEY_ID="YOUR_AWS_ACCESS_KEY_ID"
export AWS_SECRET_ACCESS_KEY="YOUR_AWS_SECRET_ACCESS_KEY"
```
*(Note: For local testing, if an `artifact` folder with a local model exists, the application will intelligently fall back to the local model to bypass AWS credential requirements).*

### 4. Run the Application
```bash
python app.py
```
Access the application dashboard at: **http://localhost:5000**

---

## ⚙️ CI/CD Automation
This project implements Continuous Integration and Continuous Deployment (CI/CD) via **GitHub Actions**.

Whenever code is pushed to the `main` branch:
1. **Build & Push:** GitHub Actions builds a new Docker image and pushes it to **AWS Elastic Container Registry (ECR)**.
2. **Continuous Deployment:** The self-hosted runner on the **AWS EC2 instance** pulls the latest image from ECR, stops the old container, and spins up the new container automatically.
3. **Port Access:** The application is exposed on port `5080` (e.g., `http://<ec2-public-ip>:5080`).

**Required GitHub Secrets:**
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_DEFAULT_REGION`
- `ECR_REPO_URI`
