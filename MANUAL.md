# Manual Setup Guide for Temporal with Docker Compose

This guide explains how to set up Temporal using Docker Compose with the standard Temporal server image (not auto-setup), focusing on a single Temporal service, detailed database schema initialization, and comprehensive configuration explanation.

## Table of Contents
- [Introduction to Temporal](#introduction-to-temporal)
- [Temporal Architecture](#temporal-architecture)
- [Manual Setup with Docker Compose](#manual-setup-with-docker-compose)
- [Database Schema Initialization](#database-schema-initialization)
- [Configuration Explanation](#configuration-explanation)
- [Metrics Endpoint](#metrics-endpoint)
- [Temporal UI Setup](#temporal-ui-setup)
- [Temporal CLI Tools](#temporal-cli-tools)
- [Troubleshooting](#troubleshooting)

## Introduction to Temporal

Temporal is a microservice orchestration platform that enables developers to build scalable, resilient applications. It provides durable execution for your applications, managing state, handling failures, and coordinating activities across distributed systems.

Key features:
- **Durable Execution**: Workflows continue execution even through service outages
- **Failure Handling**: Built-in retry mechanisms and error handling
- **Versioning**: Support for workflow code evolution
- **Visibility**: Observability into workflow execution
- **Scalability**: Horizontal scaling of workflow processing

## Temporal Architecture

Temporal consists of several core components that work together:

### Core Services

1. **Frontend Service**: 
   - Entry point for all client requests
   - Manages API endpoints
   - Routes requests to appropriate services

2. **History Service**:
   - Maintains workflow execution history
   - Manages workflow state transitions
   - Stores events in the persistence layer

3. **Matching Service**:
   - Manages task queues
   - Dispatches tasks to workers
   - Balances load across workers

4. **Worker Service**:
   - Manages system workflows
   - Handles internal tasks like archival and replication

### Persistence Layer

Temporal requires a database to store:
- Workflow execution history
- Task queue information
- Namespace metadata
- Visibility records (for search/query)

### Client Components

1. **Temporal UI**: Web interface for monitoring and debugging workflows
2. **Temporal CLI**: Command-line tools for interacting with Temporal
3. **SDK Clients**: Libraries for writing workflow code in various languages

### Component Interaction Flow

1. Clients submit workflow execution requests to the Frontend Service
2. Frontend Service creates history records and dispatches tasks
3. Matching Service queues tasks for workers
4. Workers process tasks and report results back
5. History Service records events and state transitions
6. Visibility stores workflow metadata for querying

## Manual Setup with Docker Compose

Below is a Docker Compose configuration for setting up Temporal manually without using the auto-setup image. This configuration assumes that PostgreSQL is already set up and running, and focuses on a single Temporal service that runs all components:

```yaml
version: '3.8'

services:
  # Temporal Schema Setup (runs once to initialize the database)
  temporal-schema-setup:
    container_name: temporal-schema-setup
    image: temporalio/server:1.27.2
    environment:
      DB: "postgresql"
      DB_PORT: "5432"
      POSTGRES_USER: "temporal"
      POSTGRES_PWD: "temporal"
      POSTGRES_SEEDS: "your-postgresql-host" # Replace with your PostgreSQL host
    networks:
      - temporal-network
    command: >
      sh -c "
      temporal-sql-tool create-database -database temporal &&
      temporal-sql-tool setup-schema -v 0.0 &&
      temporal-sql-tool update -schema-dir schema/postgresql/v96/temporal/versioned &&
      temporal-sql-tool create-database -database temporal_visibility &&
      temporal-sql-tool setup-schema -v 0.0 -database temporal_visibility -d visibility &&
      temporal-sql-tool update -schema-dir schema/postgresql/v96/visibility/versioned -database temporal_visibility
      "
    restart: on-failure

  # Temporal Server (All Services)
  temporal:
    container_name: temporal
    image: temporalio/server:1.27.2
    depends_on:
      - temporal-schema-setup
    environment:
      LOG_LEVEL: "debug,info"
    volumes:
      - ./config:/etc/temporal/config
    networks:
      - temporal-network
    ports:
      - "7233:7233" # Frontend service port
      - "8000:8000" # Metrics endpoint
    command: "temporal-server start" # Starts all services by default

  # Temporal UI
  temporal-ui:
    container_name: temporal-ui
    image: temporalio/ui:2.34.0
    depends_on:
      - temporal
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CORS_ORIGINS=http://localhost:3000
    networks:
      - temporal-network
    ports:
      - "8080:8080"

  # Temporal CLI Tools
  temporal-admin-tools:
    container_name: temporal-admin-tools
    image: temporalio/admin-tools:1.27.2-tctl-1.18.2-cli-1.3.0
    depends_on:
      - temporal
    environment:
      - TEMPORAL_CLI_ADDRESS=temporal:7233
    networks:
      - temporal-network
    stdin_open: true
    tty: true

networks:
  temporal-network:
    driver: bridge
```

## Database Schema Initialization

Database schema initialization is a critical step when setting up Temporal manually. Temporal requires two separate databases:

1. **Main Database (temporal)**: Stores workflow execution history, task queues, and other core data
2. **Visibility Database (temporal_visibility)**: Stores metadata for workflow executions to enable search and visibility features

In our Docker Compose configuration, we use the `temporal-schema-setup` service to initialize both databases. Let's break down each command in detail:

### Main Database Setup

```bash
# Create the main temporal database
temporal-sql-tool create-database -database temporal
```
This command creates a new PostgreSQL database named "temporal". If the database already exists, the command will fail, but the container will retry due to the `restart: on-failure` setting.

```bash
# Initialize the base schema at version 0.0
temporal-sql-tool setup-schema -v 0.0
```
This command initializes the base schema structure in the temporal database. The `-v 0.0` flag specifies the initial version of the schema. This creates the fundamental tables and indexes required by Temporal.

```bash
# Apply all schema updates to reach the current version
temporal-sql-tool update -schema-dir schema/postgresql/v96/temporal/versioned
```
This command applies all schema migrations from version 0.0 to the latest version (v96 in this example). The `-schema-dir` parameter points to the directory containing the versioned schema update scripts. This ensures the database schema is at the correct version for the Temporal server you're running.

### Visibility Database Setup

```bash
# Create the visibility database
temporal-sql-tool create-database -database temporal_visibility
```
Similar to the first command, this creates a separate PostgreSQL database specifically for visibility data.

```bash
# Initialize the visibility schema at version 0.0
temporal-sql-tool setup-schema -v 0.0 -database temporal_visibility -d visibility
```
This initializes the base visibility schema. The additional parameters:
- `-database temporal_visibility`: Specifies which database to use
- `-d visibility`: Indicates this is for the visibility schema (different from the main schema)

```bash
# Apply all visibility schema updates
temporal-sql-tool update -schema-dir schema/postgresql/v96/visibility/versioned -database temporal_visibility
```
This applies all visibility schema migrations to reach the latest version. The schema directory is different because visibility has its own set of schema files.

### Important Considerations

1. **Database Connectivity**: Ensure your PostgreSQL server is accessible from the Temporal containers. The `POSTGRES_SEEDS` environment variable should point to your PostgreSQL host.

2. **User Permissions**: The PostgreSQL user specified in `POSTGRES_USER` must have permissions to create databases and tables.

3. **Schema Versions**: The schema version directories (`v96` in the example) should match the version of Temporal you're using. Different Temporal versions may require different schema versions.

4. **Execution Order**: The schema setup must complete successfully before starting the Temporal server.

5. **Manual Execution**: If you prefer to run these commands manually instead of using Docker, you can install the Temporal CLI tools and run the same commands directly against your PostgreSQL server.

### Troubleshooting Schema Setup

If schema initialization fails, check:

1. PostgreSQL connection parameters (host, port, username, password)
2. PostgreSQL server logs for any errors
3. Permissions of the PostgreSQL user
4. Network connectivity between the containers and PostgreSQL

You can manually verify the schema was created correctly by connecting to PostgreSQL and examining the tables:

```bash
psql -h your-postgresql-host -U temporal -d temporal -c "\dt"
psql -h your-postgresql-host -U temporal -d temporal_visibility -c "\dt"
```

## Configuration Explanation

Temporal's configuration is divided into two main files: the server configuration (`config.yaml`) and the dynamic configuration (`dynamicconfig/config.yaml`). Let's explore each in detail:

### 1. Temporal Server Configuration (config/config.yaml)

This is the main configuration file for Temporal server. Here's a detailed explanation of the key sections:

```yaml
persistence:
  defaultStore: default
  visibilityStore: visibility
  numHistoryShards: 512  # Important: This cannot be changed after initial setup!
  datastores:
    default:  # Main database configuration
      sql:
        pluginName: "postgres"  # Database type
        databaseName: "temporal"  # Database name
        connectAddr: "your-postgresql-host:5432"  # PostgreSQL host and port
        connectProtocol: "tcp"
        user: "temporal"  # Database username
        password: "temporal"  # Database password
        maxConns: 20  # Maximum number of connections
        maxIdleConns: 20  # Maximum number of idle connections
        maxConnLifetime: "1h"  # Connection lifetime
    visibility:  # Visibility database configuration
      sql:
        pluginName: "postgres"
        databaseName: "temporal_visibility"
        connectAddr: "your-postgresql-host:5432"
        connectProtocol: "tcp"
        user: "temporal"
        password: "temporal"
        maxConns: 10
        maxIdleConns: 10
        maxConnLifetime: "1h"

global:
  membership:
    maxJoinDuration: "30s"
    broadcastAddress: "127.0.0.1"  # Address for membership gossip protocol
  pprof:
    port: 7936  # Port for Go profiling
  metrics:
    prometheus:
      timerType: "histogram"  # Type of timer metric
      listenAddress: "0.0.0.0:8000"  # Metrics endpoint

services:
  frontend:  # Frontend service configuration
    rpc:
      grpcPort: 7233  # gRPC port for client connections
      membershipPort: 6933  # Membership gossip port
      bindOnLocalHost: false  # Bind to all interfaces, not just localhost
    metrics:
      prometheus:
        timerType: "histogram"
        listenAddress: "0.0.0.0:8001"  # Frontend metrics endpoint

  matching:  # Matching service configuration
    rpc:
      grpcPort: 7235
      membershipPort: 6935
      bindOnLocalHost: false
    metrics:
      prometheus:
        timerType: "histogram"
        listenAddress: "0.0.0.0:8002"  # Matching metrics endpoint

  history:  # History service configuration
    rpc:
      grpcPort: 7234
      membershipPort: 6934
      bindOnLocalHost: false
    metrics:
      prometheus:
        timerType: "histogram"
        listenAddress: "0.0.0.0:8003"  # History metrics endpoint

  worker:  # Worker service configuration
    rpc:
      grpcPort: 7239
      membershipPort: 6939
      bindOnLocalHost: false
    metrics:
      prometheus:
        timerType: "histogram"
        listenAddress: "0.0.0.0:8004"  # Worker metrics endpoint

clusterMetadata:  # Cluster configuration
  enableGlobalNamespace: false  # Enable multi-cluster namespaces
  failoverVersionIncrement: 10
  masterClusterName: "active"
  currentClusterName: "active"
  clusterInformation:
    active:
      enabled: true
      initialFailoverVersion: 0
      rpcAddress: "localhost:7233"

dcRedirectionPolicy:
  policy: "noop"  # Data center redirection policy

archival:  # Archival configuration
  history:
    state: "disabled"  # Can be "enabled" to archive workflow histories
    enableRead: false
    provider:
      filestore:
        fileMode: "0666"
        dirMode: "0766"
  visibility:
    state: "disabled"  # Can be "enabled" to archive visibility records
    enableRead: false
    provider:
      filestore:
        fileMode: "0666"
        dirMode: "0766"

namespaceDefaults:  # Default settings for new namespaces
  archival:
    history:
      state: "disabled"
      URI: ""
    visibility:
      state: "disabled"
      URI: ""

dynamicConfigClient:  # Dynamic configuration settings
  filepath: "config/dynamicconfig/config.yaml"
  pollInterval: "10s"  # How often to check for config changes
```

#### Key Configuration Parameters

1. **numHistoryShards**: This is one of the most critical settings. It determines how many shards Temporal uses to partition workflow history. This value **cannot be changed** after initial setup without data migration. For production, choose a value based on your expected scale (512-4096 is common).

2. **persistence**: Configures database connections. Ensure the `connectAddr` points to your PostgreSQL host.

3. **metrics.prometheus.listenAddress**: The endpoint where Temporal exposes Prometheus metrics.

4. **services**: When running all services in a single container, all these services will be started. The ports must not conflict.

5. **clusterMetadata**: Important for multi-cluster setups. For a single cluster, the default values work fine.

6. **archival**: Enables archiving of workflow histories and visibility records. Useful for long-term storage of completed workflows.

### 2. Dynamic Configuration (config/dynamicconfig/config.yaml)

Dynamic configuration allows you to change certain settings without restarting Temporal. Here are the most important settings:

```yaml
# Maximum length for IDs (workflow ID, run ID, etc.)
limit.maxIDLength:
  - value: 255
    constraints: {}

# Forces refresh of search attributes cache on read (development only)
system.forceSearchAttributesCacheRefreshOnRead:
  - value: true # Dev setup only. Don't use in production.
    constraints: {}

# Enables client version checking
frontend.enableClientVersionCheck:
  - value: true
    constraints: {}

# Maximum queries per second for history persistence
history.persistenceMaxQPS:
  - value: 3000
    constraints: {}

# Maximum queries per second for frontend persistence
frontend.persistenceMaxQPS:
  - value: 3000
    constraints: {}

# Number of connections for frontend history manager
frontend.historyMgrNumConns:
  - value: 10
    constraints: {}

# Rate limit for throttled logs
frontend.throttledLogRPS:
  - value: 20
    constraints: {}

# Number of connections for history manager
history.historyMgrNumConns:
  - value: 50
    constraints: {}

# Enables Kafka-based visibility (for advanced setups)
system.enableVisibilityToKafka:
  - value: false
    constraints: {}
```

#### Dynamic Configuration Format

Each setting follows this format:
```yaml
setting.name:
  - value: value_here
    constraints: {}  # Optional constraints for when this value applies
```

You can add constraints to apply different values based on namespace, task queue, or other attributes:

```yaml
# Example with constraints
frontend.persistenceMaxQPS:
  - value: 3000  # Default value
    constraints: {}
  - value: 5000  # Higher value for a specific namespace
    constraints:
      namespaceId: "namespace-uuid-here"
```

## Metrics Endpoint

Temporal exposes metrics via Prometheus-compatible endpoints that can be scraped by your existing Prometheus setup:

1. When running all services in a single container, Temporal exposes a combined metrics endpoint at port 8000 (configured in `global.metrics.prometheus.listenAddress`).

2. Additionally, each internal service also exposes its own metrics endpoint:
   - Frontend: 8001
   - History: 8003
   - Matching: 8002
   - Worker: 8004

3. These endpoints expose metrics in Prometheus format and can be directly scraped by your existing Prometheus server.

4. Important metrics to monitor include:
   - `temporal_workflow_task_schedule_to_start_latency` - Time between task scheduling and execution
   - `temporal_workflow_task_execution_latency` - Time to execute workflow tasks
   - `temporal_workflow_success_counter` - Count of successful workflow completions
   - `temporal_workflow_failed_counter` - Count of failed workflows
   - `temporal_workflow_timeout_counter` - Count of timed-out workflows
   - `temporal_persistence_latency` - Database operation latency

5. To configure your Prometheus server to scrape these endpoints, add the following to your Prometheus configuration:

```yaml
scrape_configs:
  - job_name: 'temporal'
    scrape_interval: 5s
    static_configs:
      - targets: ['temporal:8000']  # Combined metrics endpoint
```

## Temporal UI Setup

The Temporal UI is configured to connect to the Temporal server:

```yaml
temporal-ui:
  container_name: temporal-ui
  image: temporalio/ui:2.34.0
  environment:
    - TEMPORAL_ADDRESS=temporal:7233  # Points to the single Temporal service
    - TEMPORAL_CORS_ORIGINS=http://localhost:3000
  networks:
    - temporal-network
  ports:
    - "8080:8080"
```

The UI connects to the Temporal server's frontend service, which runs on port 7233. Access the UI at http://localhost:8080 after starting the services.

The UI provides several useful features:
1. Viewing and searching workflows by various criteria
2. Examining workflow execution history
3. Viewing task queue details
4. Managing namespaces
5. Debugging and troubleshooting workflow executions

## Temporal CLI Tools

The admin tools container provides access to the Temporal CLI tools. These tools are essential for managing and interacting with Temporal:

```bash
# Connect to the admin tools container
docker exec -it temporal-admin-tools sh

# Use tctl (legacy CLI) to interact with Temporal
tctl namespace list                           # List all namespaces
tctl namespace describe <namespace>           # Show namespace details
tctl workflow list -n <namespace>             # List workflows in a namespace
tctl workflow describe -w <workflow-id> -r <run-id> -n <namespace>  # Show workflow details
tctl workflow show -w <workflow-id> -r <run-id> -n <namespace>      # Show workflow history

# Use temporal CLI (newer CLI) for more commands
temporal operator namespace list              # List namespaces
temporal workflow list -n <namespace>         # List workflows
temporal workflow describe --workflow-id <workflow-id> --run-id <run-id> -n <namespace>  # Show workflow details
temporal task-queue describe --task-queue <task-queue> -n <namespace>  # Show task queue details
```

The CLI tools connect to the Temporal server's frontend service on port 7233. The connection address is configured via the `TEMPORAL_CLI_ADDRESS` environment variable in the Docker Compose file.

### Common CLI Operations

1. **Creating a Namespace**:
   ```bash
   tctl namespace register <namespace> --retention 3
   ```
   The retention flag specifies how many days to retain workflow histories (default is 3 days).

2. **Forcing Workflow Completion**:
   ```bash
   tctl workflow terminate -w <workflow-id> -r <run-id> -n <namespace> --reason "Reason for termination"
   ```

3. **Resetting Workflow**:
   ```bash
   tctl workflow reset -w <workflow-id> -r <run-id> -n <namespace> --reset-type LastWorkflowTask
   ```

4. **Viewing Workflow History**:
   ```bash
   tctl workflow show -w <workflow-id> -r <run-id> -n <namespace>
   ```

## Starting the Services

1. Create the necessary configuration directory:
   ```bash
   mkdir -p config/dynamicconfig
   ```

2. Create the configuration files as described above:
   - `config/config.yaml` - Main Temporal configuration
   - `config/dynamicconfig/config.yaml` - Dynamic configuration

3. Start the services:
   ```bash
   docker-compose up -d
   ```
   This will:
   - Run the schema setup container to initialize the databases
   - Start the Temporal server with all services
   - Start the Temporal UI
   - Start the admin tools container

4. Verify the services are running:
   ```bash
   docker-compose ps
   ```

5. Access the Temporal UI at http://localhost:8080

## Troubleshooting

### Database Connection Issues

If Temporal can't connect to PostgreSQL:
1. Verify your PostgreSQL server is running and accessible
2. Check the connection parameters in `config.yaml`:
   ```yaml
   persistence:
     datastores:
       default:
         sql:
           connectAddr: "your-postgresql-host:5432"  # Ensure this is correct
   ```
3. Verify the PostgreSQL user has the necessary permissions
4. Check Temporal logs: `docker-compose logs temporal`

### Schema Setup Failures

If database schema initialization fails:
1. Check the schema setup logs: `docker-compose logs temporal-schema-setup`
2. Verify PostgreSQL connection parameters in the Docker Compose file
3. Ensure the PostgreSQL user has permissions to create databases and tables
4. Try running the schema commands manually to see detailed error messages

### Service Startup Failures

If the Temporal server fails to start:
1. Check Temporal logs: `docker-compose logs temporal`
2. Verify configuration files are correctly mounted and formatted
3. Ensure database schema was initialized properly
4. Check for port conflicts with other services

### UI Connection Issues

If the UI can't connect to Temporal:
1. Verify the Temporal server is running: `docker-compose logs temporal`
2. Check the `TEMPORAL_ADDRESS` environment variable in the UI service
3. Ensure network connectivity between containers
4. Try accessing the Temporal API directly: `curl temporal:7233`

## Conclusion

This manual setup gives you full control over Temporal's configuration and helps understand how the different components interact. By using a single Temporal service container instead of separate services, you get a simpler deployment while still having access to all Temporal features.

For production deployments, consider additional configurations for:
1. Security (TLS, authentication)
2. High availability (multiple Temporal instances)
3. Performance tuning (adjusting resource limits, connection pools)
4. Monitoring (connecting to your existing Prometheus setup)
5. Backup and disaster recovery
