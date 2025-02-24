# Hermes Pipeline 
## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture](#architecture)
3. [Development Environment Setup](#development-environment-setup)
4. [Local Development Workflow](#local-development-workflow)
5. [CI/CD Pipeline](#cicd-pipeline)
6. [Deployment Strategies](#deployment-strategies)
7. [Data Loading Process](#data-loading-process)
8. [Hot Reloading Implementation](#hot-reloading-implementation)
9. [Troubleshooting and FAQs](#troubleshooting-and-faqs)
10. [Example Implementation](#example-implementation)

## System Overview

This documentation covers the complete architecture, development workflow, and deployment process for a Python API application that integrates with Elasticsearch. The system consists of three primary components:

1. **Python API Service**: A Flask-based REST API that serves search queries to end users
2. **Elasticsearch**: The search engine that indexes and stores our data
3. **Data Loader**: A component that populates and updates the Elasticsearch indices from external data sources

The system is designed to support rapid development through hot reloading, enabling code changes to be immediately reflected without rebuilding containers or redeploying Helm charts.

## Architecture

![Architecture Diagram](https://placeholder-for-architecture-diagram.com)

### Components

#### Python API
- Flask-based REST API
- Provides search endpoints and data processing
- Queries Elasticsearch for results
- Deployed as a Kubernetes Deployment with multiple replicas

#### Elasticsearch
- Search engine and document store
- Contains indexed data for fast retrieval
- Configured for high availability in production

#### Data Loader
- Python-based utility for importing data
- Supports both full and incremental (delta) loads
- Runs as Kubernetes Jobs (one-time) and CronJobs (scheduled)

#### Configuration Management
- Helm chart for overall application deployment
- ConfigMaps for application settings
- Secrets for sensitive credentials

## Development Environment Setup

### Prerequisites

- Docker and Docker Compose
- Kubernetes local environment (Minikube, Docker Desktop, etc.)
- Helm (version 3+)
- Skaffold
- Python 3.9+
- Git

### Initial Setup

```bash
# Clone the repository
git clone https://github.com/your-org/python-es-api.git
cd python-es-api

# Install dependencies for local development
pip install -r api/requirements.txt -r data-loader/requirements.txt

# Start local Kubernetes cluster (if not already running)
minikube start

# Deploy application in development mode
skaffold dev
```

## Local Development Workflow

### Hot Reloading Development Process

1. **Start Development Environment**:
   ```bash
   skaffold dev
   ```

2. **Make Code Changes**:
   - Edit Python files in the `api/` or `data-loader/` directories
   - Changes are automatically synced to running containers
   - Application reloads without container rebuilds

3. **View Application**:
   - API Service: http://localhost:8080/api/
   - Elasticsearch: http://localhost:9200/

4. **Testing Your Changes**:
   ```bash
   # Run API tests
   cd api && pytest

   # Run data loader tests
   cd data-loader && pytest

   # Test a specific API endpoint
   curl http://localhost:8080/api/search?q=example
   ```

### Directory Structure

```
python-es-api/
├── api/                      # Python API service
│   ├── app.py                # Main application file
│   ├── requirements.txt      # Python dependencies
│   ├── Dockerfile            # Production Docker image
│   ├── Dockerfile.dev        # Development Docker image with hot reload
│   └── tests/                # API tests
├── data-loader/              # Data loading service
│   ├── data_loader.py        # Main loader script
│   ├── requirements.txt      # Python dependencies
│   ├── Dockerfile            # Production Docker image
│   ├── Dockerfile.dev        # Development Docker image
│   └── tests/                # Loader tests
└── python-es-api/            # Helm chart for deployment
    ├── Chart.yaml            # Chart metadata
    ├── values.yaml           # Default configuration values
    └── templates/            # Kubernetes templates
        ├── api-deployment.yaml      # API Deployment
        ├── api-service.yaml         # API Service
        ├── elasticsearch-deployment.yaml
        ├── elasticsearch-service.yaml
        ├── init-job.yaml            # Initial data load Job
        ├── data-cronjob.yaml        # Scheduled delta updates
        ├── configmap.yaml           # Application config
        └── secret.yaml              # Sensitive data
```

## CI/CD Pipeline

### Pipeline Overview

Our CI/CD pipeline automates the building, testing, and deployment of the application, with support for both full deployments and hot-fix updates.

![CI/CD Pipeline Diagram](https://placeholder-for-pipeline-diagram.com)

### Pipeline Stages

1. **Build**:
   - Triggers when code is pushed to the repository
   - Builds Docker images for changed components
   - Tags images with commit SHA and 'latest'
   - Pushes images to container registry

2. **Test**:
   - Runs unit and integration tests
   - Verifies application functionality
   - Generates test coverage reports

3. **Deploy**:
   - Chooses deployment method based on changes:
     - **Full Deployment**: Uses Helm to deploy all components (for infrastructure changes)
     - **Hot Deployment**: Updates only affected components (for code-only changes)

4. **Verify**:
   - Performs health checks on deployed services
   - Verifies deployment was successful

### Pipeline Configuration

The CI/CD pipeline is defined in `.gitlab-ci.yml` (or equivalent for your CI system):

```yaml
stages:
  - build
  - test
  - deploy
  - verify

variables:
  DOCKER_REGISTRY: your-registry.example.com
  K8S_NAMESPACE: python-es-app

# Build stage jobs
build-api:
  stage: build
  script:
    - docker build -t $DOCKER_REGISTRY/python-api:$CI_COMMIT_SHA -t $DOCKER_REGISTRY/python-api:latest ./api
    - docker push $DOCKER_REGISTRY/python-api:$CI_COMMIT_SHA
    - docker push $DOCKER_REGISTRY/python-api:latest
  only:
    changes:
      - api/**/*

# Test stage jobs
test-api:
  stage: test
  script:
    - cd api
    - pip install -r requirements.txt
    - pytest
  only:
    changes:
      - api/**/*

# Deploy stage jobs
deploy-helm:
  stage: deploy
  script:
    - helm upgrade --install python-es-api ./python-es-api
      --namespace $K8S_NAMESPACE
      --set api.image.repository=$DOCKER_REGISTRY/python-api
      --set api.image.tag=$CI_COMMIT_SHA
      --set dataLoader.image.repository=$DOCKER_REGISTRY/data-loader
      --set dataLoader.image.tag=$CI_COMMIT_SHA
  only:
    - main
    - changes:
      - python-es-api/**/*

# Hot fix deployment
hotfix-api:
  stage: deploy
  script:
    - kubectl set image deployment/python-api python-api=$DOCKER_REGISTRY/python-api:$CI_COMMIT_SHA -n $K8S_NAMESPACE
    - kubectl rollout status deployment/python-api -n $K8S_NAMESPACE
  only:
    changes:
      - api/**/*
    except:
      - changes:
        - python-es-api/**/*
```

## Deployment Strategies

### Full Deployment

Used for infrastructure changes or initial setup:

```bash
# Deploy or upgrade using Helm
helm upgrade --install python-es-api ./python-es-api \
  --namespace your-namespace \
  --set api.image.repository=your-registry.example.com/python-api \
  --set api.image.tag=latest \
  --set dataLoader.image.repository=your-registry.example.com/data-loader \
  --set dataLoader.image.tag=latest
```

### Hot Deployment (API)

Used for rapid updates to the API service without changing infrastructure:

```bash
# Update only the API deployment
kubectl set image deployment/python-api python-api=your-registry.example.com/python-api:${NEW_VERSION} \
  -n your-namespace
```

### Hot Deployment (Data Loader)

Used for updating the data loader without affecting other components:

```bash
# Update the CronJob template with new image
kubectl patch cronjob python-es-api-data-update -n your-namespace \
  -p '{"spec":{"jobTemplate":{"spec":{"template":{"spec":{"containers":[{"name":"data-loader","image":"your-registry.example.com/data-loader:'${NEW_VERSION}'"}]}}}}}}'
```

## Data Loading Process

### Full Load

The full load process completely refreshes an Elasticsearch index with all available data:

1. **Execution**:
   - Runs as a Kubernetes Job after initial deployment
   - Triggered by Helm post-install hook
   - Can be manually triggered when needed

2. **Process**:
   - Creates or recreates the Elasticsearch index with proper mapping
   - Fetches all data from the source system
   - Bulk loads data into Elasticsearch
   - Optimizes the index for search performance

### Delta (Incremental) Load

The delta load process updates only changed or new data:

1. **Execution**:
   - Runs as a scheduled Kubernetes CronJob
   - Typically scheduled daily or hourly based on data change frequency

2. **Process**:
   - Identifies changes since the last run (using timestamps or change tracking)
   - Fetches only changed or new data
   - Updates existing documents and adds new documents
   - Maintains index performance with periodic optimization

### Implementation Example

```python
# data-loader/loaders/delta_load.py

import requests
import time
from elasticsearch import Elasticsearch, helpers
from datetime import datetime, timedelta

def run_delta_load(es, index_name, data_source_url, changes_since=None):
    """
    Performs an incremental (delta) load of data into Elasticsearch.
    
    Args:
        es: Elasticsearch client
        index_name: Name of the index to update
        data_source_url: URL to fetch data from
        changes_since: Time period to fetch changes for (e.g., '24h', '7d')
    """
    print(f"Starting delta load for index: {index_name}")
    start_time = time.time()
    
    # Make sure we're working with the alias, not a timestamped index
    if not es.indices.exists_alias(name=index_name):
        raise Exception(f"Index alias {index_name} does not exist. Run full load first.")
    
    # Calculate time window for changes
    if changes_since:
        # Parse the time specification (e.g., "24h", "7d")
        try:
            value = int(changes_since[:-1])
            unit = changes_since[-1]
            
            if unit == 'h':
                since_time = datetime.now() - timedelta(hours=value)
            elif unit == 'd':
                since_time = datetime.now() - timedelta(days=value)
            else:
                raise ValueError(f"Unknown time unit: {unit}")
                
            since_param = since_time.isoformat()
        except (ValueError, IndexError) as e:
            print(f"WARNING: Invalid changes_since format: {changes_since}. Using default.")
            # Default to 24 hours
            since_param = (datetime.now() - timedelta(hours=24)).isoformat()
    else:
        # Default to 24 hours
        since_param = (datetime.now() - timedelta(hours=24)).isoformat()
    
    # Fetch changed/new data
    print(f"Fetching changes since {since_param} from: {data_source_url}")
    response = requests.get(f"{data_source_url}", params={'since': since_param})
    
    if response.status_code != 200:
        raise Exception(f"Failed to fetch data: {response.status_code}")
    
    data = response.json()
    print(f"Fetched {len(data)} changed/new documents")
    
    if not data:
        print("No changes found. Delta load complete.")
        return 0
    
    # Prepare documents for bulk indexing/updating
    def generate_actions():
        for item in data:
            doc = {"_index": index_name}
            
            # If document has ID, use it for update
            if 'id' in item:
                doc["_id"] = item['id']
                # Use update action with upsert
                doc["_op_type"] = "update"
                doc["doc"] = item
                doc["doc_as_upsert"] = True
                
                # Add title suggestions for autocomplete if title exists
                if 'title' in item:
                    doc["doc"]["title_suggest"] = item['title']
            else:
                # New document, use index action
                doc["_op_type"] = "index"
                doc["_source"] = item
                
                # Add title suggestions for autocomplete if title exists
                if 'title' in item:
                    doc["_source"]["title_suggest"] = item['title']
            
            yield doc
    
    # Bulk update
    indexed_count, errors = helpers.bulk(es, generate_actions(), stats_only=False)
    
    if errors:
        print(f"WARNING: {len(errors)} errors during indexing")
        for error in errors[:5]:  # Print first 5 errors
            print(f"  - {error}")
    
    print(f"Updated {indexed_count} documents")
    
    # Force refresh to make changes visible
    es.indices.refresh(index=index_name)
    
    end_time = time.time()
    print(f"Delta load completed in {end_time - start_time:.2f} seconds")
    
    return indexed_count

### 4. API Implementation with Hot Reloading

```python
# api/app.py
from flask import Flask, jsonify, request
from elasticsearch import Elasticsearch
import os
import logging
from endpoints.search import search_bp

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

def create_app():
    """
    Application factory function to create and configure the Flask app
    """
    app = Flask(__name__)
    
    # Register blueprints
    app.register_blueprint(search_bp)
    
    # Health check endpoint
    @app.route('/api/health')
    def health_check():
        try:
            # Get Elasticsearch connection
            es_host = os.environ.get('ES_HOST', 'localhost')
            es_port = os.environ.get('ES_PORT', '9200')
            es = Elasticsearch([f'http://{es_host}:{es_port}'])
            
            # Check Elasticsearch health
            es_health = es.cluster.health()
            index_name = os.environ.get('ES_INDEX', 'my_index')
            index_exists = es.indices.exists(index=index_name)
            
            return jsonify({
                "status": "ok",
                "elasticsearch": {
                    "status": es_health['status'],
                    "cluster_name": es_health['cluster_name'],
                    "index_exists": index_exists
                },
                "version": "1.0.0"
            })
        except Exception as e:
            logger.error(f"Health check failed: {str(e)}")
            return jsonify({
                "status": "error",
                "message": str(e)
            }), 500
    
    # Debug route to demonstrate hot reloading
    @app.route('/api/debug')
    def debug_info():
        """This route is useful to test hot reloading"""
        return jsonify({
            "message": "Hot reloading is working!",
            "environment": os.environ.get('FLASK_ENV', 'production'),
            "debug": os.environ.get('FLASK_DEBUG', 'false'),
            "timestamp": datetime.now().isoformat()
        })
    
    return app

if __name__ == '__main__':
    app = create_app()
    
    # Get configuration from environment
    debug_mode = os.environ.get('FLASK_DEBUG', 'false').lower() == 'true'
    port = int(os.environ.get('PORT', 5000))
    
    # Start Flask server with hot reloading in debug mode
    app.run(host='0.0.0.0', port=port, debug=debug_mode)
```

### 5. Development Docker Setup for Hot Reloading

```dockerfile
# api/Dockerfile.dev
FROM python:3.9-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Install development dependencies
RUN pip install --no-cache-dir flask-cors pytest pytest-cov

# Set environment for development
ENV FLASK_APP=app.py
ENV FLASK_ENV=development
ENV FLASK_DEBUG=1
ENV PYTHONUNBUFFERED=1

# Keep container running and watching for changes
CMD ["python", "-m", "flask", "run", "--host=0.0.0.0", "--port=5000", "--reload"]
```

### 6. Skaffold Configuration for Hot Reloading

```yaml
# skaffold.yaml
apiVersion: skaffold/v2beta29
kind: Config
metadata:
  name: python-es-api-dev
build:
  artifacts:
  - image: your-registry.example.com/python-api
    context: api
    docker:
      dockerfile: Dockerfile.dev
    sync:
      infer:
        - '**/*.py'
        - '**/*.json'
        - '**/*.yaml'
  - image: your-registry.example.com/data-loader
    context: data-loader
    docker:
      dockerfile: Dockerfile.dev
    sync:
      infer:
        - '**/*.py'
        - '**/*.json'
        - '**/*.yaml'
deploy:
  helm:
    releases:
    - name: python-es-api-dev
      chartPath: ./python-es-api
      valuesFiles:
      - ./local-values.yaml
      setValues:
        api.image.repository: your-registry.example.com/python-api
        api.image.tag: latest
        api.env.FLASK_ENV: development
        api.env.FLASK_DEBUG: "true"
        dataLoader.image.repository: your-registry.example.com/data-loader
        dataLoader.image.tag: latest
portForward:
- resourceType: Service
  resourceName: python-api
  port: 5000
  localPort: 8080
- resourceType: Service
  resourceName: elasticsearch
  port: 9200
  localPort: 9200
```

### 7. Development Values Configuration

```yaml
# local-values.yaml
# Development environment configuration
api:
  replicaCount: 1
  env:
    FLASK_ENV: development
    FLASK_DEBUG: "true"
    APP_DEBUG: "true"
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 100m
      memory: 128Mi

elasticsearch:
  replicaCount: 1
  env:
    discovery.type: single-node
    ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    xpack.security.enabled: "false"
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 200m
      memory: 512Mi

dataLoader:
  initJob:
    enabled: true
    args: ["--mode=partial", "--index=dev_index", "--changes-since=24h"]
  cronJob:
    enabled: false  # Disable scheduled jobs during development
```

## Troubleshooting and FAQs (continued)

### Common Issues (continued)

#### Data Loader Script Errors

**Problem**: Data loader jobs fail with Python errors.

**Solutions**:
- Check logs: `kubectl logs job/python-es-api-init-data -n your-namespace`
- Verify data source URL is accessible from the cluster
- Check environment variables are correctly set in the job
- Verify Elasticsearch connection details

#### Hot Reloading in Kubernetes

**Problem**: Changes are not reflected when developing in Kubernetes.

**Solutions**:
- Ensure Skaffold is syncing files: `skaffold dev --verbosity=debug`
- Check for file permission issues
- Verify Flask debug mode is enabled
- Check if file changes are detected by watching container logs: `kubectl logs -f deploy/python-api -n your-namespace`

### FAQ (continued)

**Q: How can I optimize Elasticsearch for better search performance?**

A: Several approaches:
1. Add custom analyzers in your index mapping
2. Use more specific field types (keyword vs text)
3. Configure sharding based on data volume
4. Add field-specific boosting in your search queries
5. Implement proper caching strategies

**Q: How do I troubleshoot slow search queries?**

A: Use the Elasticsearch Explain API:
```bash
curl -X POST "localhost:9200/my_index/_explain/document_id" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "title": "search term"
    }
  }
}'
```

**Q: Can I have multiple environments with different data?**

A: Yes, create separate indices for each environment:
- Configure different index names in environment variables
- Use Helm values to set environment-specific configurations
- Implement data filtering based on environment

**Q: How can I back up my Elasticsearch data?**

A: Use Elasticsearch snapshots:
1. Register a snapshot repository
2. Create scheduled snapshot CronJobs
3. Store snapshots in cloud storage (S3, GCS)

## Summary

This documentation provides a comprehensive guide to developing and deploying a Python API with Elasticsearch using a modern CI/CD approach with hot reloading capabilities. The key components are:

1. **Development Environment**: 
   - Fast iteration with hot reloading
   - Skaffold for file syncing
   - Development-specific configurations

2. **CI/CD Pipeline**:
   - Automated builds and tests
   - Two deployment paths: full Helm deployment and hot deployment
   - No need to rebuild Helm charts for code-only changes

3. **Data Loading**:
   - Full load for initial data population
   - Delta load for incremental updates
   - Proper index management with aliases

4. **Kubernetes Deployment**:
   - Helm charts for configuration management
   - Environment-specific values
   - Resource management and scaling

By following this approach, you can streamline your development process, ship bug fixes quickly, and maintain a robust production environment with minimal downtime.

## Additional Resources

- [Elasticsearch Documentation](https://www.elastic.co/guide/index.html)
- [Flask Documentation](https://flask.palletsprojects.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Helm Documentation](https://helm.sh/docs/)
- [Skaffold Documentation](https://skaffold.dev/docs/)

## License

This documentation and example code are provided under the MIT License.

---

*Last Updated: February 24, 2025*
loader/data_loader.py
import argparse
import os
import requests
from datetime import datetime, timedelta
from elasticsearch import Elasticsearch

def parse_args():
    parser = argparse.ArgumentParser(description='Load data into Elasticsearch')
    parser.add_argument('--mode', choices=['full', 'partial'], required=True, 
                        help='Full or partial data load')
    parser.add_argument('--index', required=True, 
                        help='Elasticsearch index name')
    parser.add_argument('--changes-since', 
                        help='For partial mode: only get changes since this time (e.g. 24h)')
    return parser.parse_args()

def connect_elasticsearch():
    es_host = os.environ.get('ES_HOST', 'localhost')
    es_port = os.environ.get('ES_PORT', '9200')
    return Elasticsearch([f'http://{es_host}:{es_port}'])

def get_data_source_url():
    return os.environ.get('DATA_SOURCE_URL', 'https://api.example.com/data')

def create_index_if_not_exists(es, index_name):
    if not es.indices.exists(index=index_name):
        mapping = {
            "mappings": {
                "properties": {
                    "title": {"type": "text"},
                    "content": {"type": "text"},
                    "tags": {"type": "keyword"},
                    "updated_at": {"type": "date"},
                    "created_at": {"type": "date"}
                }
            }
        }
        es.indices.create(index=index_name, body=mapping)
        print(f"Created index {index_name} with mapping")

def full_load(es, index_name, data_url):
    print("Running FULL data load")
    
    # Optionally recreate index for clean start
    if es.indices.exists(index=index_name):
        es.indices.delete(index=index_name)
        print(f"Deleted existing index {index_name}")
    
    create_index_if_not_exists(es, index_name)
    
    # Fetch all data
    response = requests.get(data_url)
    data = response.json()
    
    # Bulk insert
    bulk_data = []
    for item in data:
        # Add document ID if exists, otherwise let ES generate one
        index_action = {
            "index": {
                "_index": index_name
            }
        }
        if 'id' in item:
            index_action["index"]["_id"] = item['id']
            
        bulk_data.append(index_action)
        bulk_data.append(item)
    
    if bulk_data:
        es.bulk(body=bulk_data)
    
    print(f"Loaded {len(data)} documents in full mode")

def partial_load(es, index_name, data_url, changes_since=None):
    print("Running PARTIAL data load")
    
    create_index_if_not_exists(es, index_name)
    
    # Build query parameters for partial data
    params = {}
    if changes_since:
        # Parse time specification (e.g., "24h")
        value = int(changes_since[:-1])
        unit = changes_since[-1]
        
        if unit == 'h':
            since_time = datetime.now() - timedelta(hours=value)
        elif unit == 'd':
            since_time = datetime.now() - timedelta(days=value)
        
        params['since'] = since_time.isoformat()
    
    # Fetch partial data
    response = requests.get(data_url, params=params)
    data = response.json()
    
    # Bulk update/insert
    bulk_data = []
    for item in data:
        if 'id' in item:
            # Update existing document
            bulk_data.append({
                "update": {
                    "_index": index_name,
                    "_id": item['id']
                }
            })
            bulk_data.append({
                "doc": item,
                "doc_as_upsert": True  # Create if doesn't exist
            })
        else:
            # Insert new document
            bulk_data.append({
                "index": {
                    "_index": index_name
                }
            })
            bulk_data.append(item)
    
    if bulk_data:
        es.bulk(body=bulk_data)
    
    print(f"Loaded {len(data)} documents in partial mode")

def main():
    args = parse_args()
    es = connect_elasticsearch()
    data_url = get_data_source_url()
    
    if args.mode == 'full':
        full_load(es, args.index, data_url)
    elif args.mode == 'partial':
        partial_load(es, args.index, data_url, args.changes_since)
    
    print("Data loading complete")

if __name__ == "__main__":
    main()
```

## Hot Reloading Implementation

Hot reloading allows developers to see code changes reflected immediately without container rebuilds, dramatically speeding up the development process.

### API Hot Reloading

The API service uses Flask's built-in reloading capability for development:

1. **Development Dockerfile** (`api/Dockerfile.dev`):
   ```dockerfile
   FROM python:3.9-slim
   
   WORKDIR /app
   
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   
   # Development only dependencies
   RUN pip install --no-cache-dir watchdog[watchmedo]
   
   # Set environment for hot reloading
   ENV FLASK_ENV=development
   ENV FLASK_DEBUG=1
   ENV PYTHONUNBUFFERED=1
   
   COPY . .
   
   # Use Flask's built-in reloader
   CMD ["python", "-m", "flask", "run", "--host=0.0.0.0", "--port=5000", "--reload"]
   ```

2. **Application Code** (`api/app.py`):
   ```python
   from flask import Flask, request, jsonify
   from elasticsearch import Elasticsearch
   import os
   
   app = Flask(__name__)
   
   # Configuration from environment variables
   es_host = os.environ.get('ES_HOST', 'localhost')
   es_port = os.environ.get('ES_PORT', '9200')
   es_index = os.environ.get('ES_INDEX', 'my_index')
   
   # Connect to Elasticsearch
   es = Elasticsearch([f'http://{es_host}:{es_port}'])
   
   @app.route('/api/search')
   def search():
       """
       Search endpoint that queries Elasticsearch
       """
       query = request.args.get('q', '')
       size = int(request.args.get('size', 10))
       from_param = int(request.args.get('from', 0))
       
       if not query:
           return jsonify({"error": "Query parameter 'q' is required"}), 400
       
       # Build Elasticsearch query
       search_query = {
           "query": {
               "multi_match": {
                   "query": query,
                   "fields": ["title^2", "content", "tags"],
                   "fuzziness": "AUTO"
               }
           },
           "size": size,
           "from": from_param
       }
       
       # Execute search
       try:
           results = es.search(index=es_index, body=search_query)
           return jsonify(results['hits'])
       except Exception as e:
           app.logger.error(f"Search error: {str(e)}")
           return jsonify({"error": "Search failed"}), 500
   
   @app.route('/api/health')
   def health():
       """
       Health check endpoint
       """
       try:
           # Verify Elasticsearch connection
           health = es.cluster.health()
           return jsonify({
               "status": "healthy",
               "elasticsearch": health['status']
           })
       except Exception as e:
           return jsonify({
               "status": "unhealthy",
               "error": str(e)
           }), 500
   
   if __name__ == '__main__':
       debug_mode = os.environ.get('FLASK_DEBUG', 'false').lower() == 'true'
       app.run(host='0.0.0.0', port=5000, debug=debug_mode)
   ```

### Data Loader Hot Reloading

The data loader leverages Skaffold's file sync capability for development:

1. **Skaffold Configuration** (`skaffold.yaml`):
   ```yaml
   apiVersion: skaffold/v2beta29
   kind: Config
   metadata:
     name: python-es-api-dev
   build:
     artifacts:
     - image: your-registry.example.com/data-loader
       context: data-loader
       docker:
         dockerfile: Dockerfile.dev
       sync:
         infer:
           - '**/*.py'
           - '**/*.json'
   ```

2. **Development Values** (`local-values.yaml`):
   ```yaml
   dataLoader:
     env:
       DEBUG: "true"
     initJob:
       enabled: true
       args: ["--mode=partial", "--index=dev_index", "--changes-since=24h"]
   ```

## Troubleshooting and FAQs

### Common Issues

#### Hot Reloading Not Working

**Problem**: Code changes are not reflected in the running application.

**Solutions**:
- Verify Skaffold is running in dev mode: `skaffold dev`
- Check file sync patterns in `skaffold.yaml`
- Ensure Flask debug mode is enabled in the container
- Restart Skaffold if needed: `skaffold dev --cleanup=false`

#### Elasticsearch Connection Issues

**Problem**: API cannot connect to Elasticsearch.

**Solutions**:
- Check Elasticsearch service is running: `kubectl get pods`
- Verify Elasticsearch is healthy: `kubectl port-forward svc/elasticsearch 9200:9200` then `curl localhost:9200/_cluster/health`
- Ensure correct environment variables in API deployment

#### CI/CD Pipeline Failures

**Problem**: CI/CD pipeline fails during deployment.

**Solutions**:
- Check for syntax errors in the changed files
- Verify container builds successfully locally
- Ensure Kubernetes namespace exists
- Check credentials for container registry and Kubernetes cluster

### FAQ

**Q: How can I manually trigger a data load?**

A: You can create a one-time job from the CronJob:
```bash
kubectl create job --from=cronjob/python-es-api-data-update manual-update -n your-namespace
```

**Q: How do I change the Elasticsearch index schema?**

A: Update the mapping in `data_loader.py` and run a full load job with index recreation:
```bash
kubectl create job --from=cronjob/python-es-api-data-update schema-update -n your-namespace -- --mode=full --index=my_index --recreate-index
```

**Q: How can I monitor the data load progress?**

A: View the logs of the running job:
```bash
kubectl logs -f job/manual-update -n your-namespace
```

**Q: Can I develop without Kubernetes locally?**

A: Yes, you can use Docker Compose for a simpler setup:
```yaml
# docker-compose.yml
version: '3'
services:
  api:
    build:
      context: ./api
      dockerfile: Dockerfile.dev
    ports:
      - "5000:5000"
    volumes:
      - ./api:/app
    environment:
      - ES_HOST=elasticsearch
      - ES_PORT=9200
      - ES_INDEX=my_index
      - FLASK_DEBUG=1
    depends_on:
      - elasticsearch
      
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.14.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data
      
volumes:
  esdata:
```
