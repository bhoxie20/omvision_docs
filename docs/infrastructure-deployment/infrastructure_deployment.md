# 2. Infrastructure & Deployment

OMVision runs as a containerized application deployed on AWS ECS (Elastic Container Service). The infrastructure is designed around Dagster's multi-service architecture, where job orchestration, execution, and business logic are separated into distinct containers that communicate via gRPC. This section describes how OMVision is packaged, deployed, and run across local development and production environments.

The deployment strategy balances operational simplicity with production reliability: Docker Compose manages service definitions for both local and cloud environments, Poetry handles Python dependency management, and Alembic tracks database schema evolution. Understanding this infrastructure is essential for deploying updates, troubleshooting runtime issues, and extending the system's capacity.

---

## 2.1 Docker Architecture

OMVision uses a **three-service Docker architecture** based on Dagster's recommended deployment pattern for production systems. This separation of concerns allows the Dagster UI, job execution engine, and user code to scale and fail independently.

**[Diagram Placeholder: Docker Service Architecture]**

*Suggested diagram: Three containers (dagster_webserver, dagster_daemon, deal_flow_code) connected via gRPC and shared PostgreSQL, with arrows showing gRPC communication and job queue flow*

```mermaid
graph TB
    subgraph Docker["Docker Compose Environment"]
        DW["dagster_webserver<br/>Port: 3000<br/>0.5 CPU | 2GB RAM"]
        DD["dagster_daemon<br/>Job Orchestrator<br/>0.5 CPU | 2GB RAM"]
        DC["deal_flow_code<br/>Code Server<br/>0.5 CPU | 2GB RAM"]
        
        DB[("PostgreSQL<br/>Port: 5432<br/>Metadata Store")]
    end
    
    %% gRPC connections
    DW -->|gRPC| DC
    DD -->|gRPC| DC
    
    %% Database connections
    DW --> DB
    DD --> DB
    DC --> DB
    
    %% Styling
    classDef webStyle fill:#e3f2fd,stroke:#1976D2,stroke-width:2px
    classDef daemonStyle fill:#f3e5f5,stroke:#7B1FA2,stroke-width:2px
    classDef codeStyle fill:#fff3e0,stroke:#F57C00,stroke-width:2px
    classDef dbStyle fill:#e8f5e9,stroke:#388E3C,stroke-width:3px
    
    class DW webStyle
    class DD daemonStyle
    class DC codeStyle
    class DB dbStyle
```

### 2.1.1 Service Composition

The system comprises three containerized services defined in `docker-compose.yaml`, each with a specific role in the orchestration workflow:

### dagster_webserver

The **Dagster webserver** provides the Dagit UI, a web-based interface for monitoring jobs, viewing run history, and inspecting pipeline execution. The webserver is stateless—it does not contain any user-defined job code or execute jobs directly.

**Key Characteristics:**

- **Purpose**: Serves the Dagit UI on port 3000 for job visualization and manual triggering
- **Image**: Built from `DockerfileDagster` and pushed to ECR as `dealflow/webserver`
- **Entrypoint**: `dagster-webserver -h 0.0.0.0 -p 3000 -w workspace.yaml`
- **Workspace Configuration**: Loads job definitions from the gRPC server specified in `workspace.yaml` (points to `deal_flow_code` service on port 4000)
- **Job Submission**: When a user manually triggers a job via Dagit, the run is enqueued to the `QueuedRunCoordinator` and later executed by the daemon
- **Dependencies**: Depends on `deal_flow_code` service being available for gRPC communication

**Resource Limits:**

- CPU: 0.5 cores
- Memory: 2GB

**IAM Permissions** (via `x-aws-role`):

- `ecs:DescribeTasks` and `ecs:StopTask`: Monitor and stop ECS tasks
- `iam:PassRole`: Pass IAM roles to ECS tasks during job execution

### dagster_daemon

The **Dagster daemon** is a long-running background service responsible for executing scheduled jobs, processing sensors, and dequeuing runs from the coordinator. It is the core execution engine of the system.

**Key Characteristics:**

- **Purpose**: Dequeues runs from the `QueuedRunCoordinator` and launches them as ECS tasks using the `EcsRunLauncher`
- **Image**: Built from `DockerfileDagster` and pushed to ECR as `dealflow/daemon`
- **Entrypoint**: `dagster-daemon run`
- **Restart Policy**: `on-failure` ensures the daemon restarts if it crashes
- **Job Execution**: When a scheduled job or sensor fires, the daemon creates a new ECS task running the `deal_flow_code` image with the specific job configuration
- **Dependencies**: Depends on `deal_flow_code` service for gRPC communication to load job definitions

**Resource Limits:**

- CPU: 0.5 cores
- Memory: 2GB

**IAM Permissions** (via `x-aws-role`):

- `ecs:*` actions: Describe, register, run, and tag ECS tasks
- `ec2:DescribeNetworkInterfaces`: Query VPC networking for task placement
- `secretsmanager:*`: Retrieve secrets for injecting credentials into job runs
- `iam:PassRole`: Pass IAM roles to newly launched ECS tasks

**Design Note**: The daemon does not execute job logic itself—it orchestrates ECS task creation. Each job runs in its own isolated container using the `deal_flow_code` image.

### deal_flow_code

The **deal_flow_code** service is a **gRPC server** that exposes OMVision's job definitions, resources, and business logic to the webserver and daemon. It contains all user-defined code (jobs, ops, assets, schedules, sensors) and serves as the authoritative source for pipeline definitions.

**Key Characteristics:**

- **Purpose**: Runs a gRPC server on port 4000 that serves job metadata and code locations to other Dagster services
- **Image**: Built from `DockerfileAppCode` and pushed to ECR as `dealflow/code`
- **Code Location**: Defined in `workspace.yaml` as `DealFlow` location at `deal_flow_code:4000`
- **Restart Policy**: `always` ensures high availability—if the gRPC server crashes, the container restarts immediately
- **Job Execution**: When the daemon launches a job, it spins up a new ECS task using this same `deal_flow_code` image, but with a different entrypoint that executes the specific job rather than running the gRPC server

**Resource Limits:**

- CPU: 0.5 cores
- Memory: 2GB

**Environment Variable**: The `DAGSTER_CURRENT_IMAGE` environment variable is set to the `deal_flow_code` image URI, instructing the `EcsRunLauncher` to use this image for job execution tasks.

**Design Rationale**: Separating the gRPC server from job execution allows:

- The code location to remain available even during long-running jobs
- Jobs to run in isolated containers with independent resource limits
- Multiple code locations to be added in the future without redeploying the daemon or webserver

---

### 2.1.2 Container Configurations

All three services share common configuration patterns defined in `docker-compose.yaml`:

### Platform & Image Registry

- **Platform**: All services specify `platform: linux/amd64` to ensure compatibility with AWS ECS (which uses x86_64 EC2 instances)
- **Image Registry**: Images are stored in AWS ECR (Elastic Container Registry) under the `010526247397.dkr.ecr.us-east-1.amazonaws.com/dealflow/` namespace
- **Image Tags**: Images use the `latest` tag in production; version tagging is not currently implemented

### Network Configuration

- **Docker Network**: All services connect to a shared bridge network (`docker_network`) to enable inter-container communication
- **Service Discovery**: Containers reference each other by service name (e.g., `deal_flow_code`) which Docker resolves to internal IP addresses
- **Port Exposure**: Only the webserver exposes port 3000 to the host for UI access; the daemon and code server communicate internally via the Docker network

### Resource Limits

Resource limits are defined in the `deploy` section to prevent any single service from exhausting host resources:

| Service | CPU Limit | Memory Limit |
| --- | --- | --- |
| dagster_webserver | 0.5 cores | 2GB |
| dagster_daemon | 0.5 cores | 2GB |
| deal_flow_code | 0.5 cores | 2GB |

These limits apply to both local development (Docker Desktop enforces them loosely) and production ECS tasks (where they are strictly enforced).

**Design Note**: The 2GB memory limit is sufficient for the gRPC server and daemon. Individual job runs launched by the daemon inherit these limits but can be overridden via ECS task definitions if specific jobs require more resources.

### Inter-Service Communication

The three services communicate as follows:

1. **Webserver → Code Server (gRPC)**: The webserver queries the code server to load job definitions, asset metadata, and pipeline structures for display in Dagit
2. **Daemon → Code Server (gRPC)**: The daemon queries the code server to load schedules, sensors, and job definitions for execution
3. **Daemon → ECS API**: When executing a job, the daemon calls the AWS ECS API to launch a new task running the `deal_flow_code` image
4. **All Services → PostgreSQL**: All services connect to a shared PostgreSQL instance for storing run history, event logs, schedules, and system state

**[Diagram Placeholder: Service Communication Flow]**

*Suggested diagram: Webserver and Daemon both connecting to Code Server via gRPC, Daemon connecting to AWS ECS API, all three connecting to PostgreSQL*

---

### 2.1.3 Image Registry (ECR)

OMVision's Docker images are stored in **AWS Elastic Container Registry (ECR)**, a private container registry that integrates with ECS for secure image deployment.

### ECR Repository Structure

Three ECR repositories are configured under the AWS account `010526247397`:

| Repository | Purpose | Dockerfile | Update Frequency |
| --- | --- | --- | --- |
| `dealflow/webserver` | Dagster UI service | `DockerfileDagster` | Rarely (only on Dagster upgrades) |
| `dealflow/daemon` | Job execution coordinator | `DockerfileDagster` | Rarely (only on Dagster upgrades) |
| `dealflow/code` | User-defined jobs and business logic | `DockerfileAppCode` | Frequently (on every code change) |

### Image Build Process

Images are built locally using Docker Compose and pushed to ECR manually. The typical workflow is:

1. **Build images locally**: `docker-compose build --no-cache` compiles all three images using their respective Dockerfiles
2. **Authenticate with ECR**: `aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 010526247397.dkr.ecr.us-east-1.amazonaws.com`
3. **Push images to ECR**: Push the updated images (typically only `dealflow/code` changes frequently)

**Dockerfile Details** (inferred from docker-compose.yaml):

- **DockerfileDagster**: Installs Dagster webserver and daemon packages, copies `dagster.yaml` and `dagster-prod.yaml` configuration files. Used for both webserver and daemon images.
- **DockerfileAppCode**: Installs application dependencies via Poetry, copies the entire `app/` directory containing jobs and resources. Used for the code server and job execution containers.

### Image Versioning & Deployment

Currently, all images use the `latest` tag, meaning each push overwrites the previous version. This simplifies deployment but prevents rollbacks to previous versions.

**Operational Note**: When deploying code changes, only the `dealflow/code` image needs to be rebuilt and pushed, since job logic lives exclusively in this image. The webserver and daemon images rarely change unless Dagster itself is upgraded.

---

## 2.2 AWS ECS Deployment

OMVision runs in production on **AWS ECS (Elastic Container Service) with Fargate**, a serverless container orchestration platform. ECS manages task scheduling, health monitoring, and automatic restarts, eliminating the need to provision or manage EC2 instances.

**[Diagram Placeholder: ECS Deployment Workflow]**

*Suggested diagram: Developer pushes code → Build & push to ECR → Docker context update → ECS pulls images → Tasks running in Fargate*

```mermaid
sequenceDiagram
    participant Dev as Developer<br/>(Local Machine)
    participant Docker as Docker Build
    participant ECR as AWS ECR<br/>(Image Registry)
    participant CF as CloudFormation<br/>(IaC)
    participant ECS as AWS ECS<br/>(Fargate)
    participant Tasks as Running Tasks<br/>(Containers)
    
    Dev->>Docker: 1. docker-compose build --no-cache
    Docker-->>Dev: Images built locally
    
    Dev->>Dev: 2. Export AWS credentials
    Dev->>ECR: 3. aws ecr get-login-password
    ECR-->>Dev: Authentication token
    
    Dev->>ECR: 4. docker push dealflow/webserver:latest
    Dev->>ECR: 5. docker push dealflow/daemon:latest
    Dev->>ECR: 6. docker push dealflow/code:latest
    ECR-->>ECR: Images stored in registry
    
    Dev->>CF: 7. docker --context dealflow compose up
    CF->>CF: Convert docker-compose.yaml<br/>to CloudFormation templates
    
    CF->>ECS: Create/Update Task Definitions
    CF->>ECS: Create/Update ECS Services
    
    ECS->>ECR: Pull latest images
    ECR-->>ECS: dealflow/webserver:latest
    ECR-->>ECS: dealflow/daemon:latest
    ECR-->>ECS: dealflow/code:latest
    
    ECS->>Tasks: Launch Fargate Tasks
    Tasks-->>Tasks: dagster_webserver running
    Tasks-->>Tasks: dagster_daemon running
    Tasks-->>Tasks: deal_flow_code running
    
    Tasks-->>Dev: Deployment complete<br/>Services available at port 3000
    
    Note over Dev,Tasks: Deployment Time: 3-5 minutes<br/>Rolling update strategy (zero downtime)
```

### 2.2.1 ECS Task Definitions

ECS **task definitions** are generated automatically by Docker Compose when deploying to the ECS context. Each service in `docker-compose.yaml` maps to an ECS task definition with the following properties:

### Task Definition Configuration

- **Launch Type**: Fargate (serverless—no EC2 instance management required)
- **Network Mode**: `awsvpc` (each task gets its own elastic network interface and private IP)
- **CPU & Memory**: Inherited from `docker-compose.yaml` deploy limits (0.5 vCPU, 2GB RAM per task)
- **Execution Role**: IAM role with permissions to pull images from ECR and retrieve secrets from Secrets Manager
- **Task Role**: IAM role with permissions defined in `x-aws-role` sections of `docker-compose.yaml` (ECS task management, secret retrieval)

### IAM Roles & Permissions

ECS tasks require two types of IAM roles:

**Execution Role** (used by ECS agent):

- `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`: Pull Docker images from ECR
- `logs:CreateLogStream`, `logs:PutLogEvents`: Write container logs to CloudWatch
- `secretsmanager:GetSecretValue`: Retrieve environment variables stored in AWS Secrets Manager

**Task Role** (used by containerized application):

- Permissions specified in `x-aws-role` sections of `docker-compose.yaml`
- `dagster_webserver`: Permissions to describe and stop ECS tasks (for run management via UI)
- `dagster_daemon`: Permissions to register, run, and tag ECS tasks (for launching job execution containers)
- Both roles include `iam:PassRole` to allow passing roles to newly created tasks

**Security Note**: Task roles follow the principle of least privilege—the webserver cannot launch new tasks, and the daemon cannot access secrets beyond what's needed for job execution.

### Networking & Security Groups

ECS tasks run in a VPC with the following networking configuration:

- **Subnets**: Tasks are placed in private subnets with egress-only internet access via a NAT Gateway
- **Security Groups**: Inbound traffic is restricted to:
    - Port 3000 (Dagit UI) accessible from VPN or bastion host
    - Port 4000 (gRPC) accessible only within the VPC for inter-service communication
- **Database Access**: Tasks connect to an RDS PostgreSQL instance in the same VPC via security group rules

---

### 2.2.2 Manual Deployment Process

OMVision uses a **manual deployment workflow** based on Docker Compose's ECS integration. Deployment is triggered by engineers pushing updated images to ECR and updating the ECS stack.

### Deployment Steps

The deployment process (documented in `README.md`) consists of the following steps:

**1. Create Docker ECS Context** (one-time setup):

```bash
docker context create ecs dealflow

```

This creates a Docker context named `dealflow` that maps `docker compose` commands to ECS API calls.

**2. Build Images Locally**:

```bash
docker-compose build --no-cache

```

Compiles all three images (`webserver`, `daemon`, `code`) from their respective Dockerfiles. The `--no-cache` flag ensures clean builds without stale layers.

**3. Authenticate with AWS**:

```bash
export AWS_ACCESS_KEY_ID="<access-key>"
export AWS_SECRET_ACCESS_KEY="<secret-key>"
export AWS_SESSION_TOKEN="<session-token>"

```

Temporary credentials are retrieved from the AWS console (assuming MFA-enabled IAM users). These credentials are valid for the duration of the session.

**4. Login to ECR**:

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 010526247397.dkr.ecr.us-east-1.amazonaws.com

```

Authenticates Docker CLI with ECR using a temporary password generated by the AWS CLI.

**5. Push Images to ECR**:

```bash
docker push 010526247397.dkr.ecr.us-east-1.amazonaws.com/dealflow/daemon:latest
docker push 010526247397.dkr.ecr.us-east-1.amazonaws.com/dealflow/code:latest
docker push 010526247397.dkr.ecr.us-east-1.amazonaws.com/dealflow/webserver:latest

```

Pushes all three images to ECR. In practice, only `dealflow/code` is pushed regularly, as the other images rarely change.

**6. Update ECS Stack**:

```bash
docker --context dealflow compose --project-name dealflow up

```

This command:

- Converts `docker-compose.yaml` to CloudFormation templates
- Submits the templates to AWS CloudFormation
- CloudFormation creates or updates ECS task definitions, services, load balancers, and networking resources
- ECS pulls the `latest` images from ECR and restarts tasks with the new code

**Deployment Time**: The stack update typically takes 3-5 minutes. CloudFormation performs a rolling update, starting new tasks before stopping old ones to minimize downtime.

### Operational Notes

- **No CI/CD Pipeline**: Deployment is currently manual. Engineers trigger deployments from their local machines after verifying changes in local development.
- **Rollback Procedure**: Since images use the `latest` tag, rollback requires re-pushing a previous image version to ECR and re-running the stack update.
- **Deployment Verification**: After deployment, engineers verify the update by:
    1. Checking CloudFormation stack status in the AWS Console
    2. Accessing Dagit UI at `http://<load-balancer-dns>:3000` to confirm the webserver is running
    3. Monitoring CloudWatch Logs for any startup errors in the daemon or code server
    4. Verifying that scheduled jobs execute successfully

---

### 2.2.3 Environment Variables

OMVision requires numerous environment variables for database connections, API keys, and AWS resource identifiers. These variables are defined in a `.env.prod` file that is referenced by `docker-compose.yaml` via `env_file: ".env.prod"`.

**Security Note**: The `.env.prod` file is never committed to version control. In production, sensitive values are stored in AWS Secrets Manager and injected into ECS tasks via the `secrets` field in task definitions.

### Required Environment Variables

The following categories of environment variables are required for OMVision to function:

**Dagster Storage** (PostgreSQL for run history, event logs, schedules):

- `DAGSTER_LOGS_PG_USERNAME`: PostgreSQL username
- `DAGSTER_LOGS_PG_PASSWORD`: PostgreSQL password
- `DAGSTER_LOGS_PG_HOST`: PostgreSQL hostname (RDS endpoint)
- `DAGSTER_LOGS_PG_DB`: Database name for Dagster metadata (separate from application data)

**Application Database**:

- `SYSTEM_DB_CONN_STRING`: SQLAlchemy connection string for the OMVision application database (e.g., `postgresql://user:pass@host:5432/omvision`)

**External API Keys**:

- `OPENAI_API_KEY`: OpenAI API key for NER and context extraction
- `GOOGLE_SEARCH_API_KEY`: Google Custom Search API key
- `GOOGLE_SEARCH_ENGINE_ID`: Custom Search Engine ID
- `HARMONIC_API_KEY`: Harmonic API key for company enrichment
- `GMAIL_SERVICE_ACCOUNT_KEY`: Base64-encoded Google service account JSON for Gmail API access

**AWS Resources** (for S3 IO Manager):

- `S3_BUCKET_NAME`: S3 bucket for intermediate data storage
- `S3_BUCKET_REGION`: AWS region for the S3 bucket
- `S3_ACCESS_KEY_ID`: IAM access key with S3 permissions
- `S3_SECRET_ACCESS_KEY`: IAM secret key

**Dagster Runtime**:

- `DAGSTER_CURRENT_IMAGE`: Full ECR image URI for `deal_flow_code` (e.g., `010526247397.dkr.ecr.us-east-1.amazonaws.com/dealflow/code:latest`)

### Configuration Files

In addition to environment variables, Dagster's behavior is controlled by YAML configuration files:

- **dagster.yaml**: Defines storage backends (PostgreSQL for run storage, event logs, and schedules) used in both local and production environments
- **dagster-prod.yaml**: Production-specific configuration including:
    - `DagsterDaemonScheduler`: Enables schedule execution via the daemon
    - `QueuedRunCoordinator`: Queues runs for asynchronous execution
    - `EcsRunLauncher`: Launches jobs as ECS Fargate tasks
- **workspace.yaml**: Maps code locations to gRPC servers (points to `deal_flow_code:4000`)

**Configuration Hierarchy**: Dagster merges `dagster.yaml` and `dagster-prod.yaml` at runtime, with production-specific settings overriding defaults.

---

## 2.3 Local Development Setup

Engineers developing OMVision run the system locally using Poetry for dependency management, Docker for PostgreSQL, and Dagster's CLI for job execution. The local environment mirrors production architecture but uses local containers instead of ECS tasks.

### 2.3.1 Poetry Environment

OMVision uses **Poetry** (a Python dependency and packaging manager) to manage project dependencies and virtual environments. Poetry simplifies dependency resolution and ensures reproducible builds across developer machines.

### Installation & Setup

Prerequisites:

- Python 3.9–3.12 (specified in `pyproject.toml`)
- Poetry installed globally (`pip install poetry` or via official installer)

**Setup Steps**:

1. **Activate Poetry Shell**:

```bash
poetry shell

```

Creates and activates a virtual environment isolated from system Python packages.

1. **Install Dependencies**:

```bash
poetry install

```

Reads `pyproject.toml` and installs all dependencies (Dagster, SQLAlchemy, OpenAI SDK, etc.) into the virtual environment. The `poetry.lock` file ensures all developers use identical dependency versions.

### Key Dependencies

From `pyproject.toml`:

- **Orchestration**: `dagster`, `dagster-webserver`, `dagster-postgres` (core orchestration framework and storage backend)
- **Database**: `sqlalchemy`, `alembic` (ORM and migrations)
- **Data Processing**: `pandas`, `dagster-pandas` (dataframe operations and Dagster integration)
- **ML & NLP**: `lightgbm`, `mord`, `openai`, `tiktoken` (classification model, ordinal regression, OpenAI API)
- **APIs**: `google-api-python-client`, `accern-data` (external integrations)
- **Utilities**: `pydantic`, `fuzzywuzzy`, `ratelimit`, `tenacity` (validation, string matching, rate limiting, retries)

**Development Tools**: `black` (code formatting), `ipykernel` (Jupyter notebook support for exploration)

---

### 2.3.2 Database Setup

Local development requires a PostgreSQL instance for both Dagster metadata storage and OMVision application data. The simplest setup uses Docker to run a local PostgreSQL container.

### Local PostgreSQL Configuration

**Option 1: Docker Compose** (recommended):
Add a `postgres` service to `docker-compose.yaml` (or use a separate `docker-compose.local.yaml`):

```yaml
postgres:
  image: postgres:14
  container_name: omvision_postgres
  environment:
    POSTGRES_USER: omvision
    POSTGRES_PASSWORD: devpassword
    POSTGRES_DB: omvision
  ports:
    - "5432:5432"
  volumes:
    - postgres_data:/var/lib/postgresql/data

```

**Option 2: Managed PostgreSQL** (e.g., local Postgres.app or Homebrew installation).

### Alembic Migrations

OMVision uses **Alembic** to version control database schema changes. Migrations are stored in the `alembic/versions/` directory and applied via the Alembic CLI.

**Applying Migrations** (after installing dependencies and starting PostgreSQL):

```bash
alembic upgrade head

```

This command:

1. Reads `alembic.ini` to locate migration scripts
2. Connects to the database specified in `SYSTEM_DB_CONN_STRING` (from `.env` file)
3. Applies all unapplied migrations in chronological order
4. Updates the `alembic_version` table to track the current schema version

**Migration Files**: Each migration is a Python script in `alembic/versions/` (e.g., `5615fc35a13a_phase_one_schema.py`, `ee99f2c24b09_add_indexes_for_company_table.py`). These files are generated via `alembic revision --autogenerate` and reviewed manually before committing.

**Environment Configuration**: The `alembic/env.py` file:

- Loads the `SYSTEM_DB_CONN_STRING` from environment variables
- Imports SQLAlchemy models from `app.db.base` to enable autogeneration
- Configures online and offline migration modes

---

### 2.3.3 Running Locally

After setting up the Poetry environment and database, engineers run OMVision locally using a custom `make dev` command (defined in a Makefile, though the Makefile itself is not present in the repository snapshot).

### Local Development Workflow

**Start the Development Environment**:

```bash
make dev

```

This command (inferred from README.md) likely:

1. Starts a local PostgreSQL container via Docker Compose
2. Launches Dagster's `dagster dev` command, which starts:
    - Dagster webserver (Dagit UI) on port 3000
    - Dagster daemon for executing schedules and sensors
    - gRPC server hosting job definitions from `app/main.py`

**Accessing Dagit**:

Once running, engineers access the Dagit UI at `http://localhost:3000` to:

- View job definitions and schedules
- Manually trigger jobs for testing
- Inspect run logs and debug failures
- View asset lineage and data dependencies

### Local vs. Production Differences

| Aspect | Local Development | Production (ECS) |
| --- | --- | --- |
| **Orchestration** | Single process (`dagster dev`) | Three separate ECS tasks |
| **Job Execution** | Runs in the same process as the daemon | Runs in isolated ECS tasks |
| **Configuration** | Uses `dagster.yaml` only | Uses `dagster.yaml` + `dagster-prod.yaml` |
| **Run Launcher** | In-process execution | `EcsRunLauncher` (Fargate tasks) |
| **Database** | Local PostgreSQL (Docker or native) | AWS RDS PostgreSQL |
| **Secrets** | `.env` file | AWS Secrets Manager |

**Design Note**: The local environment executes jobs in-process for faster iteration, while production isolates each job in its own container for fault isolation and resource control.

### Development Best Practices

- **Test Migrations Locally**: Always run `alembic upgrade head` after pulling schema changes to ensure local database matches production
- **Use `.env` for Secrets**: Copy `.env.example` to `.env` and populate with development API keys (never commit `.env` to Git)
- **Verify Jobs Before Deploying**: Manually trigger jobs in local Dagit and inspect logs before pushing code to production
- **Monitor Resource Usage**: Jobs that work locally may OOM (out of memory) in production due to 2GB container limits—test with production-scale data when possible

---

**Next Section**: §3 Data Pipeline & Processing Logic (see `omvision_doc_structure.txt` for full outline)