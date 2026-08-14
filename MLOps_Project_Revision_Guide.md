# MLOps Project Revision Guide: Vehicle Insurance Data Pipeline

## 1. Project Overview & Workflow
This project demonstrates an end-to-end MLOps pipeline for predicting customer response to a vehicle insurance offer. It handles everything from data ingestion to model deployment using a robust and automated CI/CD pipeline.

### Pipeline Workflow:
1. **Data Ingestion**: Data is fetched from MongoDB Atlas, converted into a pandas DataFrame, and passed to the next stages.
2. **Data Validation**: The data schema is validated against predefined rules in `config/schema.yaml`.
3. **Data Transformation**: Feature engineering and pre-processing are applied (handling missing values, scaling, encoding).
4. **Model Training**: A machine learning model is trained, evaluated, and encapsulated via custom estimators.
5. **Model Evaluation**: The trained model is evaluated based on its metrics. If its score is better than the existing model in production by a certain threshold, it gets approved for production.
6. **Model Deployment (Pusher)**: The approved model is pushed to an AWS S3 bucket which acts as a model registry.
7. **CI/CD Automation**: When new code is pushed to GitHub, GitHub Actions trigger a workflow that builds a Docker image, pushes it to AWS ECR, and deploys it on an AWS EC2 instance.
8. **Web Application**: A FastAPI web interface allows users to submit vehicle details and get real-time predictions.

### Key Project Files & Directories:
- `app.py`: The main FastAPI entry point for the web application, handling inference (`/`) and training routes (`/train`).
- `requirements.txt`: Lists all Python dependencies (e.g., pandas, fastapi, pymongo, boto3, scikit-learn).
- `setup.py` & `pyproject.toml`: Configures the project as a local Python package, allowing imports like `from src.components import ...`. `pyproject.toml` manages modern project metadata and links dynamically to `requirements.txt`.
- `Dockerfile`: Blueprint for containerizing the FastAPI application.
- `.github/workflows/aws.yaml`: The CI/CD script that automates the Docker build, ECR push, and EC2 pull steps on AWS.
- `src/components/`: Contains the core ML pipeline steps (`data_ingestion.py`, `data_transformation.py`, `model_trainer.py`, etc.).
- `src/pipeline/`: Contains end-to-end pipelines (`training_pipeline.py` and `prediction_pipeline.py`).
- `config/`: Contains YAML files for configurations and data schema validations.

---

## 2. Tech Stack Justification & Alternatives

### Why This Tech Stack? (Our Choices)
- **MongoDB Atlas (Database)**: A NoSQL database that stores data in a flexible JSON-like format (BSON). **Why?** It's highly scalable, scheme-less, and easy to set up in the cloud (Atlas). Great for fast ingestion of structured and semi-structured tabular data.
- **AWS S3 (Model Registry)**: Cloud object storage. **Why?** It is cheap, highly available, and easily integrated with Python via `boto3`. Perfect for saving `.pkl` model artifacts and retrieving them during inference.
- **FastAPI (Web Framework)**: A modern web framework for building APIs with Python. **Why?** It uses `async/await` for fast, non-blocking asynchronous requests, has automatic documentation (Swagger UI), and is faster and more modern than Flask/Django for building lightweight ML APIs.
- **Docker (Containerization)**: **Why?** Solves the "it works on my machine" problem. It packages the app and all its dependencies into a single container that runs uniformly on any OS (like our EC2 Ubuntu server).
- **GitHub Actions (CI/CD)**: **Why?** Deeply integrated with GitHub repositories. Easily triggered by Git events (push/pull request) without having to manage a separate standalone Jenkins server.
- **AWS EC2 & ECR (Deployment)**: EC2 acts as our virtual machine to run the Docker container. ECR stores our Docker images privately. **Why?** AWS provides a robust ecosystem and a generous free tier for learning and small deployments, making it an industry standard.

### Alternatives & When to Use Them
- **Database Alternatives**:
  - *PostgreSQL/MySQL*: Use when data requires complex relationships, strict schemas, and ACID transactions.
  - *AWS S3 / GCP Cloud Storage (Data Lake)*: Use when dealing with massive datasets (TBs) where a database is too slow or expensive for raw data storage.
- **Model Registry Alternatives**:
  - *MLflow / Weights & Biases*: Use when you need to track hundreds of experiments, hyperparameter tuning runs, and model versions with a dedicated UI. S3 is a "poor man's" registry compared to MLflow.
- **Web Framework Alternatives**:
  - *Flask*: Use if you want a simpler, synchronous framework and don't need high-concurrency performance.
  - *Django*: Use if your ML app is part of a massive web application requiring built-in auth, admin panels, and an ORM.
- **CI/CD Alternatives**:
  - *Jenkins*: Use for highly customized, on-premise, or complex enterprise build pipelines.
  - *GitLab CI*: Use if your code is hosted on GitLab.
- **Deployment Alternatives**:
  - *Kubernetes (EKS) / AWS ECS*: Use when you need to auto-scale multiple containers across clusters based on heavy traffic. EC2 is great for a single instance, but hard to scale horizontally manually.
  - *AWS SageMaker / Vertex AI*: Use if you want a fully managed service that handles model serving, auto-scaling, and monitoring without dealing with raw EC2 instances and Dockerfiles manually.

---

## 3. Minor Points, Learnings, & Potential Problems

### Minor Technical Points
- **`async` and `await`**: FastAPI utilizes asynchronous programming. When receiving form data (`await self.request.form()`), the app doesn't block other users from sending requests. It temporarily pauses the function until data is received, improving concurrency.
- **CORS (Cross-Origin Resource Sharing)**: We configured `allow_origins=["*"]` in FastAPI. This is crucial if a separate frontend (like React on a different port) tries to call our FastAPI backend. Without CORS middleware, browsers block these cross-origin requests.
- **`pyproject.toml` vs `setup.py`**: Modern Python packaging prefers `pyproject.toml` for metadata and tool configuration, dynamically linking to `requirements.txt`. It replaces the heavy reliance on `setup.py` for metadata configuration.

### Potential Problems & Troubleshooting
1. **GitHub Self-Hosted Runner Stopping**: If you run `./run.sh` on EC2, the runner stops if you close the SSH terminal. **Fix**: Install it as a background service using `sudo ./svc.sh install` and `sudo ./svc.sh start`.
2. **MongoDB Connection Errors (IP Whitelisting)**: By default, MongoDB blocks external access. **Fix**: Ensure network access in MongoDB Atlas is set to `0.0.0.0/0` (allow all IPs) or specifically whitelist the EC2 instance's IP.
3. **Environment Variables**: Forgetting to set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, or `MONGODB_URL` will cause pipeline failures. **Fix**: Set them via terminal (`export VAR=...`), Windows system settings, or GitHub Secrets for CI/CD.
4. **Port Not Open on EC2**: The app might be running successfully in Docker, but you can't access it via browser. **Fix**: Ensure the EC2 Security Group's Inbound Rules allow Custom TCP on port `5080` (or whichever port Docker maps to) from `0.0.0.0/0`.
5. **Dependency Conflicts**: Installing packages globally can ruin environments. **Fix**: Always use a virtual environment (`conda create -n vehicle python=3.10 -y`).

---

## 4. Comprehensive Interview Questions & Answers

**Q1: Can you explain the overall architecture of your MLOps pipeline?**
> **A:** The pipeline starts with data ingestion from a MongoDB database, followed by automated data validation against a YAML schema. The data is then transformed, and a machine learning model is trained. The evaluation component checks if the new model outperforms the current model in production (stored in an AWS S3 registry). If it does, the Pusher component uploads the new model to S3. For deployment, we use GitHub Actions for CI/CD, which builds a Docker container and pushes it to AWS ECR, and then a self-hosted runner pulls and runs it on an AWS EC2 instance. The user interacts with the model via a FastAPI web interface.

**Q2: Why did you use `pyproject.toml` instead of just `setup.py`?**
> **A:** `pyproject.toml` is the modern standard (PEP 518) for Python project configurations. It centralizes metadata, dependencies, and build system requirements. It separates the build logic from the project metadata, making it cleaner and more flexible than putting everything into an executable `setup.py` script.

**Q3: How did you manage data drift or ensure data quality before training?**
> **A:** We implemented a Data Validation component that checks incoming data against a predefined schema (`schema.yaml`). It ensures all expected columns are present, datatypes match, and handles missing values. If the validation fails, the pipeline halts, preventing garbage data from ruining the model.

**Q4: How does your CI/CD pipeline work?**
> **A:** I used GitHub Actions. Whenever code is pushed to the main branch, a workflow (`aws.yaml`) is triggered. It first authenticates with AWS using GitHub Secrets. Then, it builds a Docker image from my `Dockerfile` and pushes it to Amazon ECR (Elastic Container Registry). Finally, it triggers a self-hosted runner installed on an EC2 instance to pull the latest image from ECR and run the container, exposing the FastAPI app on a specific port.

**Q5: Why did you choose FastAPI over Flask?**
> **A:** FastAPI is faster because it's built on ASGI and supports asynchronous programming (`async`/`await`). This is highly beneficial for IO-bound tasks like fetching data from MongoDB or uploading models to S3. Additionally, FastAPI automatically generates interactive API documentation (Swagger/OpenAPI), and uses Pydantic for data validation, making the code much cleaner and robust.

**Q6: What is a Self-Hosted Runner in GitHub Actions and why did you need it?**
> **A:** By default, GitHub Actions run on GitHub-hosted virtual machines. A self-hosted runner is a machine you manage (in my case, an AWS EC2 instance) that executes the CI/CD jobs. I used it so that the final deployment step—pulling the Docker image and running it—happens directly on my production EC2 server, rather than trying to figure out complex SSH maneuvers from a GitHub server to my EC2 instance.

**Q7: How did you handle Model Registry and Versioning?**
> **A:** I used an AWS S3 bucket (`my-model-mlopsproj`) as a model registry. The `model_pusher.py` script uploads the serialized model (`.pkl` file) to a specific S3 path. The `s3_estimator.py` acts as a custom wrapper to save/load models. (Note: For a more advanced setup, I would discuss migrating to MLflow for better artifact versioning, but S3 works great as a lightweight artifact store).

**Q8: What happens if your model training process throws an error halfway?**
> **A:** I implemented custom exception handling and logging modules across the project. Every component is wrapped in try-except blocks. If a step fails, the custom exception captures the exact file name, line number, and error message, logging it into the `logs/` directory. This makes debugging pipelines much faster and prevents silent failures.

**Q9: What is CORS and why did you add it to your FastAPI app?**
> **A:** CORS (Cross-Origin Resource Sharing) is a browser security feature that restricts web pages from making requests to a different domain than the one that served the web page. I configured `CORSMiddleware` with `allow_origins=["*"]` to ensure that if a frontend application is hosted on a different server or port, it can successfully hit our FastAPI prediction endpoints without being blocked by the browser.

**Q10: Why did you Containerize your application with Docker?**
> **A:** Docker ensures environmental consistency. Machine Learning apps rely on highly specific versions of libraries (like scikit-learn, pandas). By containerizing the app, I ensure that it runs exactly the same way on my local machine, the GitHub Actions runner, and the production EC2 server, eliminating the "it works on my machine" problem.

**Q11: Why are you using MongoDB for tabular data? Wouldn't a SQL database be better?**
> **A:** While SQL databases are traditionally better for tabular data due to strict schemas, MongoDB was chosen for this project for its flexibility and ease of setup in a cloud environment (Atlas). It allows us to easily dump raw JSON-like payloads without worrying about migrations during early developmental phases. However, in a strict production environment with complex tabular relationships, migrating to PostgreSQL or an S3 Data Lake with Athena would be a viable alternative.
