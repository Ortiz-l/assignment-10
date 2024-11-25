# Part 3: Configure the Backend Store

By default, MLflow authentication uses a local SQLite database to store user and permission data. The default backend data store logs all metadata to the **local filesystem** under `./mlruns` . 

We can change where the metadata is stored by adding the  `--backend-store-uri` option to our `mlflow server` command:

```python
mlflow server --backend-store-uri <backend_store_server_url>
```

Since we’re using a containerized environment to deploy MLflow tracking server, every time we restart the container, we’re going to lose the experiments and runs we’ve started. We can get around that issue in a few ways:

- Create a persistent volume for our container to use for saving experiments and runs
- Store our experiments and runs in a remote database.

If you’re running a multi-node deployment, or if you’re collaborating with other data scientists you should consider using a centralized database to store the backend data. Part 3 will walk through setting this up for our MLflow deployment! 

The MLflow container image does not include required dependencies for running Postgres (or MySQL) as a backing database, so we’ll need to create a new Docker file to build an image that has the required dependencies for us to interact with a Postgres database. 

If you haven’t yet, shut down your existing container images. 

# Step 1: Create a new image for our MLflow tracking server:

1. We’ll create a new container image and install MLflow and packages required to interact with PostgrSQL and DigitalOcean Spaces. Create a `Dockerfile` in your project directory, and add the following to your `Dockerfile` :

```docker
FROM python 3.12-slim

WORKDIR /app

RUN pip install mlflow psychopg2-binary boto3
```

1. Build the container image by running:

```bash
docker build -t mlflow-docker-example -f Dockerfile .
```

1. Make sure that your container image works as expected by running the following, replacing the `s3://mlflow-bucket-prof-z` with the bucket name that hosts your MLflow artifacts:

```bash
docker run -p 8000:5000 \
mlflow-docker-example \
mlflow server \
-h 0.0.0.0 \
--default-artifact-root **s3://mlflow-bucket-prof-z** \
--serve-artifacts
```

**Note that when we’re using a remote artifact store, the tracking server may not be able to view the remote artifacts in the UI. However, we can still query our artifacts programmatically without issue.** 

# Step 2: Create a PostgreSQL Container

For local testing, we’ll use a PostgreSQL database container:

1. Create a PostgreSQL database container.
    1. Pull the image using `docker pull postgres` 
    2. Start the Postgres instance using the following command. This will create a container named `my-postgres` and assign the default password via an environment variable before running it in the background. 
        
        ```bash
        docker run --name my-postgres -e POSTGRES_PASSWORD=mysecretpassword postgres
        ```
        
    3. In your terminal output, you’ll see the IP address and port of the database:
        
        ![postgres_sql_connection.png](img/postgres_sql_connection.png)
        
    4. There are many environment variables you can pass to PostgreSQL that influence its behaviors:
        1. `POSTGRES_USER`: specifies a user with superuser privileges and a database with the same name. Postgres uses the default user when this is empty
        2. `POSTGRES_DB` : Specifies a name for your database or defaults to the `POSTGRES_USER` value when left blank
        3. `POSTGRES_HOST_AUTH_METHOD` : Controls the `auth-method` for `host` connections to `all` databases, users, and addresses.
        4. `PGDATA` : Defines another default location or subdirectory for database files
    
    Rather than pass these as flags to our `docker run` command, we can also save these variables in a `.env` file. 
    

# Step 3: Configure the MLflow Backend Store

Since we’re now working with two containers, we’re going to use `docker-compose` to manage our MLflow deployment. Stop your MLflow container and your PostgreSQL container before proceeding. 

1. Create a `docker-compose.yml` file in your working directory. Add the following to your docker-compose file:

```docker
services:

  postgres:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s
    restart: always
    container_name: postgres_db
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  tracking-server:
    build:
      context: .
      dockerfile: "${DOCKERFILE:-Dockerfile}"
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      # MLflow UI is available at http://localhost:5050
      - "5050:5000"
    command: >
      mlflow server
      --host 0.0.0.0
      --backend-store-uri postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      --default-artifact-root s3://mlflow-bucket-prof-z 
      --serve-artifacts
      --gunicorn-opts "--log-level debug"

volumes:
  postgres_data: { }
```

The important parts to note from our Docker Compose file are:

- the `--backend-store-uri` is how we are going to specify that the MLflow backend should be the Postgres database that we create as the first service in our `docker-compose` file.
- the `tracking-server` service has a `depends_on` definition. It will check to make sure that the postgres server is healthy before creating the tracking server container image.
- We’re mapping ports `5050:5000` (instead of 8000; this doesn’t really matter, I just wanted to call it out for clarity.)
- You will need to change the `--default-artifact-root` in your `docker-compose` file to point it to the Spaces bucket you created.
- We created a persistent volume called `postgres_data` and told our postgres database container to use that volume to store data.
1. Now, in your `.env` file, add the following environment variables so that they can be referenced by docker compose: 
    1. `POSTGRES_USER`
    2. `POSTGRES_PASSWORD` 
    3. `POSTGRES_DB` 
    
    For example:
    
    ```
    POSTGRES_USER=postgres_username
    POSTGRES_PASSWORD=postgres_password
    POSTGRES_DB=mlflow_db
    ```
    

1. Now, we’ll use docker-compose to create both containers as well as the volume:
    
    ```
    docker-compose up
    ```
    
2. Your tracking server will be available at [http://localhost:5050](http://localhost:5050) . Try running through an experiment again, ensure that the experiment is logged, and then press `ctrl+c` to stop your running containers. (Don’t run `docker-compose down` yet, as it will remove the volume we created.)
3. Now, run `docker-compose up` to re-create the database and server containers, and notice that the experiment and run you created are still available in the UI, even though we are using containers! 

# Step 4: Create a Managed PostgreSQL Database in DigitalOcean, and connect our local MLflow server to it

First, let’s create a new managed database in DigitalOcean cloud console:

Login to the console, then select **Create**, and **Databases:** 

![create_managed_database.png](img/create_managed_database.png)

 Select PostgreSQL, then scroll down, and select

- Basic- Shared CPU
    - CPU options: Regular
- Select a plan:
    - $13.00/month
- Storage size:
    - 10GB
- Finalize and create:
    - Choose a unique database cluster name:
        - choose something unique that you will be able to connect to. For example `db-postgresql-sfo3-prof-z`
    - Select a Project:
        - Assignment 10
    - Tags:
        - mlflow-assignment-10

Then select **Create Database Cluster.** Once the database cluster is provisioned, we’ll add a user and a database for our MLflow tracking service: 

- Navigate to the **Users & Databases** tab. Under the **Users** heading, add a new Username called `mlflow_user`
- Now, add a new database called `mlflowdb`

![user_and_database.png](img/user_and_database.png)

Once complete, navigate to the **Overview** tab. 

In the **Configuration Details** section, select `Public network`  and then `Connection string` . Select the **User: mlflow_user** and the **Database/Pool: mlflowdb:**

![connection_details.png](img/connection_details.png)

Since we’re using a remote database, we don’t need to use the Postgres container that we defined in our `docker-compose` file. Let’s modify our `docker-compose.yml` to use our remote database:

First, open your `.env` file and update your `POSTGRES_USERNAME` , `POSTGRES_PASSWORD` , and `POSTGRES_DB` environment variables to reflect the new username and database that we created in the cloud console. 

- For example, the `POSTGRES_USERNAME`  will now be mlflow_user
- The `POSTGRES_PASSWORD` can be found in the **Users & Databases** tab in the managed database console.
- The `POSTGRES_DB` will be **mlflowdb**

Save your `.env` file, and then open your `docker-compose.yml` file and make the following changes. Pay careful attention to the `backend-store-uri` and the `default-artifact-root` . Make sure that these values are the correct ones for the cloud resources you created. 

- Note also, that we are passing environment variables to our CMD. These are pulled from your .env file!

```docker
services:
# Remove the postgres service from the compose file by deleting these lines
  # postgres:
  #   image: postgres
  #   healthcheck:
  #     test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
  #     interval: 10s
  #     retries: 5
  #     start_period: 30s
  #     timeout: 10s
  #   restart: always
  #   container_name: postgres_db
  #   environment:
  #     POSTGRES_DB: ${POSTGRES_DB}
  #     POSTGRES_USER: ${POSTGRES_USER}
  #     POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
  #   volumes:
  #     - postgres_data:/var/lib/postgresql/data
  #   ports:
  #     - "5432:5432"

  tracking-server:
    build:
      context: .
      dockerfile: "${DOCKERFILE:-Dockerfile}"
      # We don't need to include the depends_on key anymore
    # depends_on:
    #   postgres:
    #     condition: service_healthy
    ports:
      # MLflow UI is available at http://localhost:5050
      - "5050:5000"
    command: >
      mlflow server
      --host 0.0.0.0
      --backend-store-uri postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db-postgresql-sfo3-14267-do-user-17404056-0.i.db.ondigitalocean.com:25060/${POSTGRES_DB}?sslmode=require
      --default-artifact-root s3://mlflow-bucket-prof-z 
      --serve-artifacts
      --gunicorn-opts "--log-level debug"
# We don't need to persist a volume anymore
# volumes:
#   postgres_data: { }
```

Now, run `docker-compose up` to launch our MLflow server with the managed database backend! 

Note: If you're having trouble connecting with the username we created, you can use the doadmin username and password. 

**To Complete Assignment 10, you will need to do the following:**

Deploy MLflow on a droplet, either using a container or installing it directly on the droplet. You will need to ensure that you’re using the managed Postgres database you just created as the backing server for your MLflow deployment.

1. Once you’ve created MLflow on a droplet, you should switch the connection details to use the `VPC network` . That will make it so that the only resources that can hit the database are from the internal network. 
2. Also ensure that you’re using the Spaces bucket you created as the remote artifacts storage for your MLflow deployment! 
3. Add a firewall rule to your droplet so that only connections from inside the VPC network can access the MLflow server.
    1. Navigate to your droplet, and then select **Firewalls**. Think about the inbound and outbound rules you would need to create to ensure that *only connections from within the VPC can access your droplet.*
- The Jupyter notebook you created in Assignment 2 can serve as the testing basis for your MLflow server deployment!

For the project submission, please add the following in D2L: 

- The name of the droplet that’s running your MLflow Tracking server
    - We will inspect the firewall rules and use the IP address to connect to the MLflow UI
- The name of the managed database that’s backing your MLflow Tracking server
- The name of the bucket that’s backing your MLflow Registered artifacts

# Best Practices for Remote DB Stores

- Utilize **connection pooling** (e.g., PgBouncer) to manage database connections efficiently
- Apply appropriate **Indexing** on the MLflow database tables to speed up query performance
- Regularly **vacuum** your PostgreSQL database to prevent transaction IP wraparound issues
- Always backup your database before running `mlflow db upgrade` to update the schema
- Store **database credentials** securely using environment variables or secret management services
- Place the MLflow server behind a reverse proxy and use VPN or secure transport layers for communication
- Use **batch logging** of parameters, metrics, and tags to reduce the number of database transactions
- Implement **asynchronous logging** to prevent blocking the main execution thread.
- Scale the MLflow tracking server horizontally by running multiple instances behind a load balancer
- Use cloud storage like S3 or Spaces for **artifact storage** for scalability and durability
- **Monitor** the MLflow server and PostgreSQL databases performance regularly
- Schedule **maintenance jobs** for tasks like backups, vacuuming, and index rebuilding.