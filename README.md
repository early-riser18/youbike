# YouBike 🚲
This project aims at improving the fleet rebalancing of YouBikes in Taiwan.
Currently, it provides a service to predict the number of bikes available at any YouBike station in Taiwan in the next three hours.
**Try it out on [Streamlit](https://smarter-youbike.streamlit.app/) now!**

## What is the problem?
YouBike users who wish to get a bike to travel around the city are sometimes not able to find any bike at the station surrounding them. Therefore they have to rely on other methods of transportation, which might be more expensive, slower or less convenient. 

## How we are solving it
As a first step, we focus on forecasting the number of YouBikes available at any stations in the next three hours.
From there, we aim to help the Youbike team make more informed decisions on which stations are most likely to run out of bikes, and therefore need to be resupplied.

Youbike stations' current state is ingested every 30 minutes, transformed and clean to form a standardized layer for downstream services.

*Disclaimer: This project is first a data engineering project. The current model is naive and does not produce good results. It will be improved in the future.*
# How it works
![Architecture Diagram](https://raw.githubusercontent.com/early-riser18/youbike/500540beb25cb0c12b56d5c967d1d7d837601b0e/assets/youbike_architecture_diagram.png)
A predictive timeseries model is trained using real-time, publicly accessible data on the quantity of bikes docked at YouBike stations, as well as weather data.
## Requesting a forecast
The predictive model is hosted on the cloud and queried via API. When requesting to forecast the number of bikes available at the Youbike stations, the real-time state of all Youbike stations is retrieved and combined with a forecast of the day's weather.
<br><br>**Try it out**
```bash
curl "https://33iqhftc6fakzdlczkjsiikhtu0odrec.lambda-url.ap-northeast-1.on.aws/" -H 'content-type: application/json' -d '{"service": "fill_rate_forecast", "station_id": [508201032, 501208101]}'
```
Note: More station ids can be found by querying the other available service.
```bash
curl "https://33iqhftc6fakzdlczkjsiikhtu0odrec.lambda-url.ap-northeast-1.on.aws/" -H 'content-type: application/json' -d '{"service": "bike_station_status", "extended": 1}'
```
*Note: The API call may fail on the first try due to the lambda cold start. If so, just try again once.* 
## Training the model
The model is retrained periodically based on the latest historical data. 

### Data sources
YouBike Stations data is obtained via [YouBike Official Website](https://www.youbike.com.tw/region/main/stations/)<br>
Weather data is obtained thanks to the [Open Meteo API](https://open-meteo.com/)<br>
 
# Set up project
Start by cloning this repository.
```bash
git clone https://github.com/early-riser18/youbike.git && cd ./youbike

# Install the python dependencies (always use a virtual env)
pip install -r requirements.txt
```
Ensure you have [Terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) & [Docker](https://docs.docker.com/engine/install/) installed.
```bash
terraform -v 
docker run hello-world
```
Make sure to set the following three variables in `terraform.variables.tf`
```tf
variable "cockroachdb_sql_user_password" {}
variable "prefect_api_key" {}
variable "cockroachdb_api_key" {}
```

## Local set up
### Set up your environment
Create env `.env` file, add the following environment variables with a filled value and source it.
```bash
### MINIO ###
export MINIO_ACCESS_KEY_ID=minio-local
export MINIO_SECRET_ACCESS_KEY=minio-local
export MINIO_HOST=localhost

### COCKROACHDB ###
export DATABASE_URL="" # Find it on your CockroachDB account. Looks like: cockroachdb://<account-name>:<password>@<project-name>-6569.6xw.<cloud-region>.cockroachlabs.cloud:26257/defaultdb?sslmode=verify-full

export APP_ENV=local #replace by 'stage' to run it on the cloud-based stage env.
```
## Cloud Environment
You will need the following accounts: Terraform, Prefect Cloud, AWS

1. Spin up the cloud infrastructure (VPC, ECR, ECS Cluster...)
    ```bash
    cd <path>/youbike/terraform
    terraform init
    terraform login
    terraform apply
    ```
3. Build an image with the Dockerfile and push it to the newly-created ECR
3. Create a Prefect Work Pool of type ECS:Push by passing the infrastructure ids generated ([see source documentation](https://docs.prefect.io/latest/guides/deployment/push-work-pools/#manual-infrastructure-provisioning))
4. Deploy the flows to your Prefect Server by running `python3 -m etl.flows.\<flowname>

# Testing
## Locally
You can test the main functionalities by running the following scripts:
- Get fill rate forecast of YouBike stations: `python3 -m api.forecast_service`
- Get latest status of YouBike stations: `python3 -m api.get_bike_station_status`
- ELT YouBike Snapshots: `python3 -m etl.flows.youbike_snapshot_ingestion`
- ELT Weather forecast & historical report: `python3 -m etl.flows.get_weather`

Make sure to set `APP_ENV=local` 
## Container
This service is run in the cloud in a container and you can test the containerized environment locally.
Build all Docker resources by running once `docker compose -f ./docker/compose.yaml build`
 
Then you can test the containerized env by running `./docker/docker_script.sh` and select the option accordingly:

`1) Test Locally`: Build image and run code in a containarized local environment
<br>`2) Test Lambda`: Build image and launch a local lambda function emulator. You can then make calls to it with `curl "http://localhost...` to simulate the cloud lamdbda function behavior. 
```bash
curl "http://localhost:9999/2015-03-31/functions/function/invocations" -H 'content-type: application/json' -d '{"body": "{\"service\": \"fill_rate_forecast\", \"station_id\": [508201032, 501208101]}"}'
```
Note: More station ids can be found by querying the other available service.
```bash
curl "http://localhost:9999/2015-03-31/functions/function/invocations" -H 'content-type: application/json' -d '{"body": "{\"service\": \"bike_station_status\", \"extended\": 1}"}'
```

# Train & Test Forecast Model
You can train the model locally on a Spark Cluster with multiple worker nodes. To do so, simply run the following command.
```bash
cd docker/
make run-stage
```
This command will launch a local Spark Cluster with three worker nodes, and register a Spark Job as specified by the entrypoint of the Docker Service "gateway". This job will retrain the model on all existing snapshot data found on s3://stage-youbike/clean-data/.