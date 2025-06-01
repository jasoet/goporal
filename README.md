# Temporal Workflow Engine with Docker Compose

This repository contains a Docker Compose configuration for running Temporal Workflow Engine locally. Temporal is a microservice orchestration platform that enables developers to build scalable, resilient applications.

## What is Temporal?

Temporal is a workflow orchestration platform that provides:

- **Durable Execution**: Workflows continue execution even through service outages
- **Failure Handling**: Built-in retry mechanisms and error handling
- **Versioning**: Support for workflow code evolution
- **Visibility**: Observability into workflow execution
- **Scalability**: Horizontal scaling of workflow processing

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Running Temporal

To start Temporal and its dependencies:

```bash
cd compose
docker-compose up -d
```

This will start the following services:

1. **PostgreSQL** - Database for Temporal (exposed on port 5434)
2. **Temporal Server** - The main Temporal service (exposed on port 7233)
3. **Temporal UI** - Web interface for monitoring workflows (exposed on port 8233)
4. **Temporal Admin Tools** - CLI tools for interacting with Temporal

## Accessing Temporal UI

Once the services are running, you can access the Temporal UI at:

```
http://localhost:8233
```

The UI allows you to:
- View and search workflows
- Examine workflow execution history
- View task queue details
- Manage namespaces
- Debug workflow executions

## Using Temporal CLI Tools

You can use the Temporal CLI tools by connecting to the admin tools container:

```bash
docker exec -it temporal-admin-tools sh
```

Common CLI commands:

```bash
# List all namespaces
temporal namespace list

# Create a new namespace with 3-day retention
temporal namespace create --retention 3d --description "My new namespace" my-namespace

# List workflows in a namespace
temporal workflow list -n default

# Show workflow details
temporal workflow describe --workflow-id <workflow-id> --run-id <run-id> -n default

# Show task queue details
temporal task-queue describe --task-queue <task-queue> -n default
```

## Configuration

The Temporal setup uses the following configuration:

- **Database**: PostgreSQL with automatic schema initialization
- **Default Namespace**: "default" with 7-day retention period
- **Metrics**: Prometheus-compatible metrics exposed on port 7244
- **Dynamic Configuration**: Minimal configuration in `compose/temporal/config.yaml`

## Stopping Temporal

To stop all services:

```bash
cd compose
docker-compose down
```

To stop and remove all data (including the PostgreSQL volume):

```bash
cd compose
docker-compose down -v
```

## Local Development with Temporal CLI

While Docker Compose is recommended for a complete development environment, you can also run Temporal locally using the CLI installed via Homebrew for a lighter-weight setup.

### Installing Temporal CLI via Homebrew

```bash
# Install Temporal CLI
brew install temporal

# Verify installation
temporal --version
```

### Starting a Local Temporal Server

```bash
# Start the Temporal server with in-memory persistence
temporal server start-dev

# Start with a specific namespace
temporal server start-dev --namespace my-namespace

# Start with UI on a custom port (default is 8233)
temporal server start-dev --ui-port 8080
```

The development server will be available at:
- Temporal Server: localhost:7233
- Web UI: localhost:8233 (default)

### Key Differences from Docker Compose Setup

- **Persistence**: The `start-dev` command uses in-memory persistence by default, so data is lost when the server stops
- **Components**: Only runs the Temporal server and UI, without separate PostgreSQL
- **Configuration**: Uses default configuration without the custom settings in `config.yaml`
- **Performance**: Lighter resource usage, suitable for local development

### Connecting to Your Local Temporal Server

```bash
# List namespaces
temporal namespace list

# Create a workflow
temporal workflow start --task-queue my-task-queue --type MyWorkflowType --input '"Hello World"' --workflow-id my-workflow

# Describe a workflow
temporal workflow describe --workflow-id my-workflow
```

## Troubleshooting

If you encounter issues:

1. Check service status: `docker-compose ps`
2. View logs: `docker-compose logs temporal`
3. Ensure all ports are available (5434, 7233, 8233, 7244)
