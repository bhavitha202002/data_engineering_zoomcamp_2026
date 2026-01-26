# Module 1 Homework Solutions: Docker & SQL

## Question 1. Understanding Docker images

Run docker with the `python:3.13` image. Use an entrypoint `bash` to interact with the container.

**What's the version of `pip` in the image?**

- 25.3
- 24.3.1
- 24.2.1
- 23.3.1

**Solution:**

```bash
docker run -it --entrypoint bash python:3.13
pip --version
```

**Answer: `24.3.1`**

---

## Question 2: Understanding Docker networking and docker-compose

**Given the following `docker-compose.yaml`, What is the hostname and port that pgadmin should use to connect to the postgres database?**

```yaml
services:
  db:
    container_name: postgres
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: 'postgres'
      POSTGRES_PASSWORD: 'postgres'
      POSTGRES_DB: 'ny_taxi'
    ports:
      - '5433:5432'
    volumes:
      - vol-pgdata:/var/lib/postgresql/data

  pgadmin:
    container_name: pgadmin
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: "pgadmin@pgadmin.com"
      PGADMIN_DEFAULT_PASSWORD: "pgadmin"
    ports:
      - "8080:80"
    volumes:
      - vol-pgadmin_data:/var/lib/pgadmin

volumes:
  vol-pgdata:
    name: vol-pgdata
  vol-pgadmin_data:
    name: vol-pgadmin_data
```

- postgres:5433
- localhost:5432
- db:5433
- postgres:5432
- db:5432

**Solution:**

Containers in the same docker-compose project are on the same internal Docker network.

They talk to each other using the service name as the hostname (db), not localhost.

Inside the Docker network, Postgres is listening on its internal port 5432.

The 5433:5432 mapping is only for access from your host machine, not from other containers.

So from pgAdmin → Postgres, the correct connection is:

**hostname: db**
**port: 5432**


**Answer: `db:5432`**

---

## Prepare the Data

Download the green taxi trips data for November 2025:

```bash
wget https://d37ci6vzurychx.cloudfront.net/trip-data/green_tripdata_2025-11.parquet
```

You will also need the dataset with zones:

```bash
wget https://github.com/DataTalksClub/nyc-tlc-data/releases/download/misc/taxi_zone_lookup.csv
```

## Question 3. Counting short trips

**For the trips in November 2025 (lpep_pickup_datetime between '2025-11-01' and '2025-12-01', exclusive of the upper bound), how many trips had a `trip_distance` of less than or equal to 1 mile?**

- 7,853
- 8,007
- 8,254
- 8,421

**Solution:**

```sql
SELECT
  COUNT(*)
FROM green_taxi_trips
WHERE
  lpep_pickup_datetime >= '2025-11-01'
  AND lpep_pickup_datetime < '2025-12-01'
  AND trip_distance <= 1;
```

**Answer: `8,007`**

---

## Question 4. Longest trip for each day

**Which was the pick up day with the longest trip distance? Only consider trips with `trip_distance` less than 100 miles (to exclude data errors).**

**Use the pick up time for your calculations.**

- 2025-11-14
- 2025-11-20
- 2025-11-23
- 2025-11-25

```sql
SELECT
  date_trunc('day', lpep_pickup_datetime) AS pickup_day,
  MAX(trip_distance) as max_distance
FROM green_taxi_trips
WHERE
  lpep_pickup_datetime >= '2025-11-01'
  AND lpep_pickup_datetime < '2025-12-01'
  AND trip_distance < 100
GROUP BY date_trunc('day', lpep_pickup_datetime) AS pickup_day,
ORDER BY max_distance DESC
LIMIT 1;
```

**Answer: `2025-11-14`** (88.03 miles)

---

## Question 5: Biggest pickup zone

**Which was the pickup zone with the largest `total_amount` (sum of all trips) on November 18th, 2025?**

- East Harlem North
- East Harlem South
- Morningside Heights
- Forest Hills

**Solution:**

```sql
SELECT
  tz.Zone as  "Pick_up_zone",
  SUM(tz.total_amount) as "total_amount"
FROM green_taxi_trips t
JOIN taxi_zones tz ON t.PULocationID = tz.LocationID
WHERE DATE(t.lpep_pickup_datetime) = '2025-11-18'
GROUP BY z.Zone
ORDER BY total DESC
LIMIT 1;
```

**Answer: `East Harlem North`** ($9,281.92)

---

## Question 6. Largest tip

**For the passengers picked up in the zone named "East Harlem North" in November 2025, which was the drop off zone that had the largest tip?**

Note: it's `tip` , not `trip`. We need the name of the zone, not the ID.

- JFK Airport
- Yorkville West
- East Harlem North
- LaGuardia Airport

**Solution:**

```sql
SELECT
  dz.Zone as dropoff_zone,
  t.tip_amount
FROM green_taxi_trips t
JOIN taxi_zones pz ON t.PULocationID = pz.LocationID
JOIN taxi_zones dz ON t.DOLocationID = dz.LocationID
WHERE pz.Zone = 'East Harlem North'
  AND t.lpep_pickup_datetime >= '2025-11-01'
  AND t.lpep_pickup_datetime < '2025-12-01'
ORDER BY t.tip_amount DESC
LIMIT 1;
```

**Answer: `Yorkville West`** ($81.89 tip)

---

## Terraform

In this section homework we'll prepare the environment by creating resources in GCP with Terraform.

In your VM on GCP/Laptop/GitHub Codespace install Terraform.
Copy the files from the course repo
[here](../../../01-docker-terraform/terraform/terraform) to your VM/Laptop/GitHub Codespace.

Modify the files as necessary to create a GCP Bucket and Big Query Dataset.

## Question 7: Terraform Workflow

**Which sequence describes the workflow for:**
1. Downloading provider plugins and setting up backend
2. Generating proposed changes and auto-executing the plan
3. Removing all resources managed by Terraform

Answers:
- terraform import, terraform apply -y, terraform destroy
- teraform init, terraform plan -auto-apply, terraform rm
- terraform init, terraform run -auto-approve, terraform destroy
- terraform init, terraform apply -auto-approve, terraform destroy
- terraform import, terraform apply -y, terraform rm

**Solution:**

Correct sequence:

- **terraform init**

Purpose:

Downloads required provider plugins (AWS, GCP, etc.)

Sets up the backend (local or remote state)

Prepares the working directory for Terraform commands

This directly matches “Downloading provider plugins and setting up backend”

- **terraform apply**

Purpose:

Runs terraform plan internally to generate proposed changes

Shows the execution plan

Automatically executes the plan after confirmation (or immediately with -auto-approve)

This matches “Generating proposed changes and auto-executing the plan”

- **terraform destroy**

Purpose:

Generates a plan to remove all resources

Deletes everything Terraform manages in that state

This matches “Removing all resources managed by Terraform”

**Answer: `terraform init, terraform apply -auto-approve, terraform destroy`**
