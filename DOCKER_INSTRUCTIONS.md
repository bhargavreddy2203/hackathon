# Docker Setup for Microservices

This project contains three microservices with Docker support:
- **patient-service** (Node.js/Express) - Port 3000
- **application-service** (Node.js/Express) - Port 3001  
- **order-service** (Java/Spring Boot) - Port 8080

## Prerequisites

- Docker Engine 20.10+
- Docker Compose 2.0+

## Quick Start

### Build and Run All Services

```bash
# Build and start all services
docker-compose up --build

# Run in detached mode
docker-compose up -d --build

# View logs
docker-compose logs -f

# Stop all services
docker-compose down
```

### Build Individual Services

```bash
# Patient Service
docker build -t patient-service:latest ./patient-service

# Application Service
docker build -t application-service:latest ./application-service

# Order Service
docker build -t order-service:latest ./order-service
```

### Run Individual Services

```bash
# Patient Service
docker run -d -p 3000:3000 --name patient-service patient-service:latest

# Application Service
docker run -d -p 3001:3001 --name application-service application-service:latest

# Order Service
docker run -d -p 8080:8080 --name order-service order-service:latest
```

## Service Endpoints

### Patient Service (Port 3000)
- Health Check: `GET http://localhost:3000/health`
- Get All Patients: `GET http://localhost:3000/patients`
- Get Patient by ID: `GET http://localhost:3000/patients/:id`
- Create Patient: `POST http://localhost:3000/patients`

### Application Service (Port 3001)
- Health Check: `GET http://localhost:3001/health`
- Get All Appointments: `GET http://localhost:3001/appointments`
- Get Appointment by ID: `GET http://localhost:3001/appointments/:id`
- Get Appointments by Patient: `GET http://localhost:3001/appointments/patient/:patientId`
- Create Appointment: `POST http://localhost:3001/appointments`

### Order Service (Port 8080)
- Health Check: `GET http://localhost:8080/actuator/health`
- Order endpoints as defined in the service

## Docker Commands Reference

### Container Management

```bash
# List running containers
docker ps

# List all containers
docker ps -a

# Stop a container
docker stop <container-name>

# Remove a container
docker rm <container-name>

# View container logs
docker logs <container-name>

# Follow container logs
docker logs -f <container-name>
```

### Image Management

```bash
# List images
docker images

# Remove an image
docker rmi <image-name>

# Remove unused images
docker image prune
```

### Docker Compose Commands

```bash
# Start services
docker-compose up

# Start services in background
docker-compose up -d

# Stop services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Rebuild services
docker-compose build

# View service logs
docker-compose logs

# Follow service logs
docker-compose logs -f

# Scale a service
docker-compose up -d --scale patient-service=3

# Check service status
docker-compose ps
```

## Testing the Services

### Using curl

```bash
# Test Patient Service
curl http://localhost:3000/health
curl http://localhost:3000/patients

# Test Application Service
curl http://localhost:3001/health
curl http://localhost:3001/appointments

# Test Order Service
curl http://localhost:8080/actuator/health
```

### Create Sample Data

```bash
# Create a patient
curl -X POST http://localhost:3000/patients \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Johnson","age":28,"condition":"Healthy"}'

# Create an appointment
curl -X POST http://localhost:3001/appointments \
  -H "Content-Type: application/json" \
  -d '{"patientId":"1","date":"2024-04-15","time":"09:00","doctor":"Dr. Williams"}'
```

## Dockerfile Details

### Node.js Services (patient-service, application-service)
- Base Image: `node:18-alpine`
- Multi-stage: No (optimized for size with alpine)
- Production dependencies only
- Non-root user: Uses default node user

### Java Service (order-service)
- Base Image: `maven:3.9-eclipse-temurin-17` (build), `eclipse-temurin:17-jre-alpine` (runtime)
- Multi-stage: Yes (reduces final image size)
- Build artifacts copied from build stage
- JRE-only runtime for smaller footprint

## Environment Variables

### Patient Service
- `PORT` - Server port (default: 3000)
- `NODE_ENV` - Environment mode (production/development)

### Application Service
- `PORT` - Server port (default: 3001)
- `NODE_ENV` - Environment mode (production/development)

### Order Service
- `SERVER_PORT` - Server port (default: 8080)
- `SPRING_PROFILES_ACTIVE` - Spring profile (prod/dev)

## Networking

All services are connected via a custom bridge network `microservices-network`, allowing inter-service communication using service names as hostnames.

```bash
# Access from one service to another
# Example: From patient-service to application-service
http://application-service:3001/appointments
```

## Health Checks

All services include health check configurations:
- **Interval**: 30 seconds
- **Timeout**: 10 seconds
- **Retries**: 3
- **Start Period**: 40-60 seconds

## Troubleshooting

### Container won't start
```bash
# Check logs
docker-compose logs <service-name>

# Inspect container
docker inspect <container-name>
```

### Port already in use
```bash
# Find process using port
lsof -i :<port-number>

# Kill process
kill -9 <PID>

# Or change port in docker-compose.yml
```

### Build failures
```bash
# Clean build
docker-compose build --no-cache

# Remove all containers and images
docker-compose down --rmi all
```

### Network issues
```bash
# Recreate network
docker-compose down
docker network prune
docker-compose up
```

## Production Considerations

1. **Security**
   - Use specific image versions instead of `latest`
   - Scan images for vulnerabilities
   - Run containers as non-root users
   - Use secrets management for sensitive data

2. **Performance**
   - Set resource limits (CPU, memory)
   - Use volume mounts for persistent data
   - Implement proper logging strategies

3. **Monitoring**
   - Add application performance monitoring (APM)
   - Implement centralized logging
   - Set up alerts for health check failures

4. **Scaling**
   - Use orchestration platforms (Kubernetes, Docker Swarm)
   - Implement load balancing
   - Use service discovery

## Clean Up

```bash
# Stop and remove all containers, networks
docker-compose down

# Remove all containers, networks, and volumes
docker-compose down -v

# Remove all images
docker-compose down --rmi all

# Complete cleanup
docker system prune -a --volumes
```
