# CryptoWatch: Distributed Event-Driven Alert System

This project is a demonstration of a microservices-based application for monitoring cryptocurrency prices and sending alerts built with **Laravel 12**. It demonstrates an Event-Carried State Transfer architecture using Kafka, Redis, and PostgreSQL.

## Overview

The CryptoWatch project is designed to fetch cryptocurrency prices from an external API and allow users to set up alerts based on price movements. When an alert is triggered, the system sends a notification to the user.

The project follows a microservices architecture, with different services responsible for specific tasks:

*   **API Gateway:** An Nginx server that acts as a reverse proxy for the backend services and serves the frontend application.
*   **Backend Services:** Three separate Laravel applications handle the core logic:
    *   `cryptowatch-api`: Manages user authentication and alerts, sends alert updates via Kafka message bus, and provides the main API.
    *   `cryptowatch-ingest`: Fetches cryptocurrency prices and ingests them into the system via a Kafka message bus.
    *   `cryptowatch-notifier`: Listens for alert events on the Kafka bus and sends notifications to users via Telegram and dispatch events to frontend via Websockets (Reverb).
*   **Frontend:** A simple HTML-based frontend for interacting with `cryptowatch-notifier` via Websockets (Reverb).
*   **Infrastructure:** The project uses a set of services for data storage, caching, messaging, and observability, all managed with Docker Compose.

## Architecture

### Event-Carried State Transfer

This project uses the **Event-Carried State Transfer** pattern. Instead of services sharing each other databases directly to get information, the `cryptowatch-ingest` service publishes price updates to a `crypto-prices` Kafka topic and the `cryptowatch-api` service publishes alerts updates also to a `alert-events` Kafka topic. Other service `cryptowatch-notifier`, subscribe to these topics and receive the data it needs. This decouples the services and improves scalability and resilience.

## Tech Stack



| Category               | Technologies                                   |
|------------------------|------------------------------------------------|
| Core                   | PHP 8.2 + Laravel 12                           |
| Auth                   | JWT                                            |
| Database               | PostgreSQL, Redis                              |
| Message Broker         | Apache Kafka                                   |
| Cache / Queue          | Redis                                          |
| Realtime               | WebSocket (Laravel Reverb)                     |
| Web Server             | Nginx                                          |
| DevOps & Observability | Docker & Docker Compose, ELK Stack, Kafka UI, Supervisor |
| Tests & Static Analysis | PEST, PHPStan                                 |


## Getting Started

To get the project up and running locally, follow these steps:

### Prerequisites

*   Git
*   Docker & Docker Compose

### Installation

1.  **Clone the repository with submodules:**

    ```bash
    git clone --recurse-submodules https://github.com/viktor-va/cryptowatch-project.git
    ```

2.  **Set up the environment file:**

    Create a `.env` file in the root of the project by copying the example file from the `cryptowatch-infrastructure` directory:

    ```bash
    cd cryptowatch-project/cryptowatch-infrastructure/
    cp .env.example .env
    ```

    You will need to create a new bot in Telegram and add a user (into that bot) who will receive notifications (won't take longer than 1 minute). Save bot token and user telegram id. After, add a `TELEGRAM_BOT_TOKEN` to the `.env` file for the notifier service. (Or just skip bot set entirely, won't break anything)


3.  **Build and run the services:**

    Use Docker Compose to build and start all the services:

    ```bash
    docker-compose up --build -d
    ```

    After all containers are up, WAIT for 2–3 minutes to ensure that the composer finishes installations in all microservice containers (`cw-api`, `cw-ingest`, `cw-notifier`)


4.  **Run database migrations:**

    Once the services are running, you need to run the database migrations for the `cryptowatch-api` service:

    ```bash
    docker-compose exec api php artisan migrate:fresh
    ```

## How to Use

### Creating Users and Alerts

You can create users and alerts by sending requests to the `cryptowatch-api` service. The API documentation is available at [http://localhost/docs/api](http://localhost/docs/api).

Here are some example `curl` commands:

**Register a new user:**

```bash
curl -X POST http://localhost/api/auth/register \
-H "Content-Type: application/json" \
-H "Accept: application/json" \
-d '{
    "name": "John Doe",
    "email": "john.doe@example.com",
    "password": "Password123"
}' | jq
```

**Log in and get a JWT token:**

```bash
curl -X POST http://localhost/api/auth/login \
-H "Content-Type: application/json" \
-H "Accept: application/json" \
-d '{
    "email": "john.doe@example.com",
    "password": "Password123"
}' | jq
```

SAVE this token, it will be valid during the next 60 minutes:
```bash
TOKEN=your-token-here
```

**Add telegram id (that you added to bot in the previous steps) to your User :**

```bash
curl -X PUT http://localhost/api/auth/me \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-H "Accept: application/json" \
-d '{"telegram_chat_id": "your-telegram-id-added-to-bot"}' | jq
```


**Create an alert:**

```bash
curl -X POST http://localhost/api/v1/alerts \
-H "Authorization: Bearer $TOKEN" \
-H "Content-Type: application/json" \
-H "Accept: application/json" \
-d '{
    "symbol": "BTC",
    "target_price": 100000,
    "condition": "gt"
}' | jq
```

### Simulating Market Data

To start the market simulation, run the following command:

```bash
docker-compose exec ingest php artisan crypto:fetch --duration=60
```

This will start a process that publishes mock cryptocurrency prices to the `crypto-prices` Kafka topic for 60 seconds. You can view the messages in the **Kafka UI** at [http://localhost:8080](http://localhost:8080). Check real-time cryptocurrency price updates at [http://localhost](http://localhost). Also, notifications will start sending to the Telegram bot if the condition of the alert is met, and you configured the bot and added the `TELEGRAM_BOT_TOKEN` to the `.env` file in the previous.


## Observability

The project uses the **ELK Stack** for logging and monitoring.

*   **Logstash:** Collects logs from all the services.
*   **Elasticsearch:** Stores and indexes the logs.
*   **Kibana:** Provides a web interface for searching and visualizing the logs.

### Correlation ID

To trace requests as they flow through the system, the application uses a **correlation ID**. When a new request comes in, a unique ID is generated and added to all log messages related to that request. This makes it easy to filter and view all the logs for a specific request in Kibana.

### How to Check Logs in Kibana

1.  Open Kibana at [http://localhost:5601](http://localhost:5601).
2.  Go to the **Data Views** page.
3.  Create data view with index pattern **cryptowatch-***
4.  Go to the **Discover** page.
5.  You can search and filter the logs using the Kibana Query Language (KQL). For example, to see all the logs for a specific correlation ID, you can use a query like:

    ```
    context.correlation_id: "your-correlation-id"
    ```

## Testing

The project uses Pest PHP with isolated containers (`cw-postgres-test` and `cw-redis-test`) to ensure running tests never wipes your development data. To run the tests, you can use the following commands:

```bash
# Run tests for cryptowatch-api
docker-compose exec api php artisan test

# Run tests for cryptowatch-notifier
docker-compose exec notifier php artisan test
```

## Static Analysis

The project uses **PHPStan** for static analysis to find potential bugs and improve code quality and is configured to run at **level 6**.
To run the static analysis for each service, use the following commands:

```bash
# Run static analysis for cryptowatch-api
docker-compose exec api ./vendor/bin/phpstan analyse --memory-limit=2G

# Run static analysis for cryptowatch-ingest
docker-compose exec ingest ./vendor/bin/phpstan analyse --memory-limit=2G

# Run static analysis for cryptowatch-notifier
docker-compose exec notifier ./vendor/bin/phpstan analyse --memory-limit=2G
```

## Summary

This project serves as a comprehensive example of a modern web application built with a microservices' architecture. It demonstrates the use of various technologies and best practices for building scalable and maintainable systems.

## License

MIT (for interview/demo use)
