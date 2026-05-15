# my2pmclub-python-socket
Chat with Python
=======

# 2PM Club Sockets

## Overview

This project is a Python-based web application built with FastAPI.

## Features

- **FastAPI**: High-performance web framework for building APIs.
- **PostgreSQL**: Relational database for storing persistent data.
- **SQLAlchemy**: ORM for interacting with the PostgreSQL database.
- **Redis**: In-memory data structure store for caching.
- **Uvicorn**: ASGI server to serve the FastAPI application.


## File Structure Overview

The project is organized into several directories, each serving a distinct purpose:

### `app/`

Contains the main API endpoints and the core business logic of the application.

- **`api.py`**: API endpoints and their associated logic.
- **`sockets.py`**: Socket events for chat functionality.

### `services/`

Contains integrations with external services and other third-party functionalities.

- **`socket_server.py`**: Initializes the SocketIO server.

### `database/`

This folder holds the database models and configurations.

- **`models.py`**: Defines the SQLAlchemy models representing the database tables.
- **`table_keys.py`**: Manages table keys and relationships.

### `utilities/`

Contains utility scripts and helper functions used across the application.

- **`config.py`**: Manages configuration settings and environment variables.
- **`constants.py`**: Defines constants used across the application.
- **`dependencies.py`**: Manages dependency injections for the application. Ex - database connection, authentication, etc.

### Root-Level Files

- **`.env`**: Environment variables configuration.
- **`main.py`**: The main entry point of the FastAPI application.
- **`requirements.txt`**: Python dependencies required for the project.
- **`runserver.py`**: Script to start the FastAPI server.

## Prerequisites

Before setting up the project, ensure you have the following installed:

- Python 3.10.12+
- PostgreSQL
- Redis

## Installation

### Clone the repository

Skip this step if the repository is already cloned.

```bash
git clone https://github.com/yourusername/yourproject.git
cd yourproject
```

### Create a virtual environment

```bash
python -m venv env
source env/bin/activate
```

### Install dependencies

```bash
pip install -r requirements.txt
```

### Set up environment variables

Create a `.env` file in the project root directory and configure the following environment variables:

Note: "Added PM keyword for parameter store"
Use env_var.py to set up environment variables in PM store.
Remove PM variables from `.env` once they are stored in PM.
Otherwise use them for development purposes.

```ini
PY_API_APP_URL=""           PM    #don't include '/' at end

PY_REDIS_URL='redis://localhost:6379/'      PM

#Database
PY_DB_HOST=         PM
PY_DB_PORT=         PM
PY_DB_NAME=         PM
PY_DB_USER=         PM
PY_DB_PASSWORD=     PM

#Server
PY_SERVER_HOST="0.0.0.0"
PY_SERVER_PORT=5000

#API Documentaion
PY_SWAGGER_USERNAME=        PM
PY_SWAGGER_PASSWORD=        PM

PY_PRODUCTION_ENV=          int (0 or 1), default 0
```

### Running the Application

To start the FastAPI application using Uvicorn, run:

```bash
python runserver.py
```

### Log Files

The application logs are stored in the `logs` directory.

- **`main_server.log`**: Logs for the main server. API calls are logged here.
- **`db_logger.log`**: Logs for database connection initiation.
- **`sockets.log`**: Logs for SocketIO events.
- **`socket_conn.log`**: Logs for user SocketIO connections.

### Accessing the API

The automatic API documentation is available at:

```
http://localhost:8000/docs/
```

The documentation is protected with basic authentication. Use the credentials set in your `.env` file to access it.
