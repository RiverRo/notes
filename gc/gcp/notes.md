Google Cloud Platform (GCP)
> [Main Table of Contents](../README.md)

## In This Notebook
- [Free Tier Program Overview](https://cloud.google.com/free/docs/free-cloud-features#free-trial)
- Google Cloud PostgreSQL
- Google Cloud PostgreSQL via Google Cloud Compute (Free Tier Settings)
- Connect to PostgreSQL instance

## Google Cloud PostgreSQL  

1. Create PostgreSQL instance
2. Example config.py
3. See "Connect to PostgreSQL instance" Section

### 1. Create PostgreSQL instance

- Google Cloud Console -> Left panel Select "SQL" -> Choose PostgreSQL flavor
- Instance info
    - Fill in instance ID
    - Fill in password (SET STRONG PASSWORD) for default admin user "postgres"
        - Aside: I can add other username & password after instance creation in the "users" tab
    - Keep latest database version selected
- Choose a configuration to start with
    - Development
- Choose region and zonal availability
    - us-central1 (Iowa)
    - Single Zone
- Customize Your instance
    - Machine type
        - Shared core -> 1 vCPU, 1.7 GB (Choose enough mem to hold largest table)
    - Storage
        - HDD -> 10GB
        - Check "Enable automatic storage increases"
    - Connections
        - Check "Public IP" (When this is check *MUST* specify allowable networks via Add Network otherwise can't use username/password instance connection option)
        - Add NetWork
            - Fill in name -> e.g. hom
            - Fill in Network -> 0.0.0.0/0  (Allow any IPv4 client to pass the network firewall and make login attempts to your instance, including clients you did not intend to allow. Clients still need valid credentials to successfully log in to your instance)
    - Data Protection
        - Check "Automate backups" -> Change time to 8:00pm-12:00am
        - Check "Enable point-in-time recovery"
        - check Enable deletion protection
    - Maintenance
        - Maintenance Window -> Change to 12:00am-1:00am
- Press Create Instance

### 2. Example config.py
```python
# config.py
PG_HOST = '999.999.999.999'  # Found in "Public IP address" section of GCP console instance overview
PG_DB = 'postgres'           # Default database auto-created with instance is "postgres"
PG_USER = 'postgres'         # Default user auto-created with instance is "postgres"
# Strong Password supplied by me in SQL instance creation settings for default "postgres" admin user (See instance info section above)
PG_PASSWORD = 'TheStrongestPasswordEVER'  
PG_PORT = '5432'             # Default PostgreSQL port
```

## Google Cloud PostgreSQL via Google Cloud Compute (Free Tier Settings)
- [Spot on step-by-step video](https://www.youtube.com/watch?v=JLdy_cJ1KRA)  

1. Create Firewall Rule
2. Create Debian VM instance in Google Cloud Compute
3. Install Docker Engine on Debian Docs - Follow this step-by-step
4. Pull in Debian Postgres Docker Image
5. Example config.py
6. See "Connect to PostgreSQL instance" Section


### 1. Create Firewall Rule
- VPC network found in left panel of GCP console -> Firewall -> +Create Firewall Rule
    - Fill in name e.g. allow-postgres
    - Fill in description e.g. Allow access to port 5432, default port for PostgreSQL
    - Logs -> Off
    - Network -> default
    - Priority - 1000
    - Direction of traffic -> Ingress
    - Action on match -> Allow
    - Targets -> Specified target tags
    - Fill in target tags e.g. allow-tcp-5432
    - Source filter -> IPv4 ranges
    - Fill in Source IPv4 ranged -> 0.0.0.0/0 (Allow any IP address)
    - Protocols and ports -> Specified protocols and ports -> Check "TCP" -> Fill in 5432 in Ports


### 2. Create Debian VM instance in Google Cloud Compute
- Compute Engine found in left panel of GCP console -> +Create Instance
    - Fill in name
    - Region must be "us-central1 (Iowa)" for free tier
    - Machine family -> General Puprose
        - Series E2
        - Machine type must be "e2-micro (2vCPU, 1GBmem) for free tier
    - Book Disk -> Change
        - Use Debian OS (will be using Debian Postgres Docker image)
        - Use latest Debian version
        - Boot Disk type must be "Standard persistent disk" for free tier
        - Size (max is 30GB in free tier) -> Change to 20GB
    - Advanced Options
        - Netowrking
            - Fill in Network tags, this is the same tag created in step 1 firewall rule.  e.g. allow-tcp-5432

### 3. SSH into VM instance server & [Install Docker Engine on Debian Docs - Follow this step-by-step](https://docs.docker.com/engine/install/debian/)

> Run everything below in sudo
- Cloud Compute instance Overview -> Find SSH connect button next to newly create VM instance
- Run every command in the "Install using the repository" section of this step's documentation link


### 4. Pull in [Debian Postgres Docker Image](https://hub.docker.com/_/postgres)
> Run everything below in sudo
- ```sudo docker pull postgres```
- ```sudo docker run --name fp_postgres  -e POSTGRES_PASSWORD=BestPasswordEver -v postgres:/var/lib/postgresql/data  -p 5432:5432 -d postgres```

    - ```--name``` Name of docker container
    - ```-e``` Set environment variables with this flag
        - e.g. Strong password.  This password is for default admin user named "postgres".  This is equivalent to the PG_PASSWORD above in the other option section
    - ```-v``` Set the volume. This is important if I want my data to persist irrespective of whether the docker is up & running or not.
    - ```-p``` Be sure to set the port of the container, both inbound:outbound to 5432 since that's what we allowed in our firewall setttings above
    - ```-d``` Use in detach mode, not sure what this means but it's in the docker doc and person in vid used it
- When done will get id of docker container
    - e.g. 554af394126d658d1d1ee7ccdcc6e635ac431faf0bcdd72111fdfc4b6631eb40


### 5. Example config
```python
# config.py
PG_HOST = '999.999.999.999'  # External IP address of Cloud compute VM instance
PG_DB = 'postgres'           # Default database auto-created with instance is "postgres"
PG_USER = 'postgres'         # Default user auto-created with instance is "postgres"
# Same password created in docker run command in step 4 instance creation settings for default "postgres" admin user (See instance info section above)
PG_PASSWORD = 'TheStrongestPasswordEVER'  
PG_PORT = '5432'             # Default PostgreSQL port
```


## Connect to PostgresQL instance
- [Many Connection Options](https://cloud.google.com/sql/docs/postgres/connect-overview?authuser=1#external-connection-methods)
    - Connect to instance through Cloud SQL Proxy (secure access to your instances without a need for Authorized networks or for configuring SSL) specifically for Python the [Cloud SQL Python Connector](https://colab.research.google.com/github/GoogleCloudPlatform/cloud-sql-python-connector/blob/main/samples/notebooks/postgres_python_connector.ipynb)
    - (PREFERRED-EASIEST) Connect through public ip with username and password 
        - (PREFERRED FOR NOW: easiest but not most secure - may have to pay for spammers?? We'll see)
        - Make sure to check "Public IP" option in the creation of an instance and to set the allowable network to 0.0.0.0/0 and create strong postgres password
        ```python
        # Connect to PostgresQL instance
        import psycopg2
        from config import PG_HOST, PG_PORT, PG_DB, PG_USER, PG_PASSWORD

        # 1. Direct Option
        conn = psycopg2.connect(host=PG_HOST, port=PG_PORT, database=PG_DB, user=PG_USER, password=PG_PASSWORD)

        # 2. Context Manager Option
        class Postgres:
            """A minimal PostgreSQL context manager that removes almost all 
            boilerplate code from the application level.
            """
            def __init__(self, *, db_name):
                self.host = PG_HOST
                self.port = PG_PORT
                self.database = db_name
                self.user = PG_USER
                self.password = PG_PASSWORD

            def __enter__(self):
                self.conn = psycopg2.connect(
                    host=self.host,
                    port=self.port,
                    database=self.database,
                    user=self.user,
                    password=self.password
                )
                self.cursor = self.conn.cursor()
                return self

            def __exit__(self, exc_type, exc_value, exc_tb):
                self.conn.commit()
                self.cursor.close()
                self.conn.close()

        # context manager usage:
        with Postgres(db_name=PG_DB) as db:
            pass
        ```