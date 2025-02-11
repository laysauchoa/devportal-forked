Aiven for Apache Kafka® as a source for Aiven for ClickHouse®
=============================================================

This article shows by way of example how to integrate Aiven for Apache Kafka® with Aiven for ClickHouse® using `Terraform provider for Aiven <https://registry.terraform.io/providers/aiven/aiven/latest/docs>`_. An Apache Kafka® source topic is used as a data source and Aiven for ClickHouse® is used to filter or transform the raw data with a materialized view before writing it to a regular table.

.. topic:: Sample of sensor data

  First, check out how sensor data can look like for a better understanding of this recipe and the ``clickhouse_kafka_user_config`` Terraform block used in this article.

  .. code-block:: shell

    {
      "sensor_id": 10000001,
      "ts": "2022-12-01T10:08:24.446369",
      "key": "cpu_usage",
      "value": 96
    }

Let's cook!
-----------

.. mermaid::

  flowchart LR
  id1(iot_measurements_topic)
  id2[(iot_measurements)]
  id3(edge_measurements_raw_table)
  id4(cpu_high_usage_table)
  subgraph Aiven for Apache Kafka
  id1
  end
  subgraph Aiven for ClickHouse
    id2
    subgraph iot_measurements DB
      id3-->|Filter|id4
    end
  end
  id1-->|Service integration|id2

Imagine that you've been collecting IoT measurements from thousands of sensors and these metrics are being populated in an Apache Kafka topic called ``iot_measurements``. Now, you'd like to set up an Aiven for ClickHouse database and write filtered messages into table ``cpu_high_usage``.

This recipe calls for the following:

1. Set up an Aiven for ClickHouse database for writing and processing raw data.
2. Insert the measurements data from Apache Kafka topic ``iot_measurements`` into the Aiven for ClickHouse database.
3. Filter the data and save the output to the new ``cpu_high_usage`` table.

Configure common files
''''''''''''''''''''''

.. dropdown:: Expand to check out the common files needed for this recipe.

  Navigate to a new folder and add the following files:

  1. ``provider.tf`` file

    .. code-block:: terraform

       terraform {
         required_providers {
           aiven = {
             source  = "aiven/aiven"
             version = "~> 3.8.1"
           }
         }
       }
   
       provider "aiven" {
         api_token = var.aiven_api_token
       }

  .. tip::

    You can set environment variable ``AIVEN_TOKEN`` for the ``api_token`` property so that you don't need to pass the ``-var-file`` flag when executing Terraform commands.

  2. ``variables.tf`` file

  Use it for defining the variables to avoid including sensitive information in source control. The ``variables.tf`` file defines the API token, the project name, and the prefix for the service name.

    .. code-block:: terraform

       variable "aiven_api_token" {
         description = "Aiven console API token"
         type        = string
       }
   
       variable "project_name" {
         description = "Aiven console project name"
         type        = string
       }

  3. ``*.tfvars`` file

  Use it to indicate the actual values of the variables so that they can be passed (with the ``-var-file=`` flag) to Terraform during runtime and excluded later on. Configure the ``var-values.tfvars`` file as follows:

    .. code-block:: terraform

       aiven_api_token     = "<YOUR-AIVEN-AUTHENTICATION-TOKEN-GOES-HERE>"
       project_name        = "<YOUR-AIVEN-CONSOLE-PROJECT-NAME-GOES-HERE>"

Configure the ``services.tf`` file
''''''''''''''''''''''''''''''''''

The following Terraform script initializes both Aiven for Apache Kafka and Aiven for ClickHouse services, creates the service integration, the source Apache Kafka topic, and the Aiven for ClickHouse database.

.. code-block:: terraform

  resource "aiven_kafka" "kafka" {
    project                 = var.project_name
    cloud_name              = "google-europe-west1"
    plan                    = "business-4"
    service_name            = "kafka-gcp-eu"
    maintenance_window_dow  = "monday"
    maintenance_window_time = "10:00:00"
  }

  resource "aiven_kafka_topic" "source" {
    project      = var.project_name
    service_name = aiven_kafka.kafka.service_name
    partitions   = 50
    replication  = 3
    topic_name   = "iot_measurements"
  }

  resource "aiven_clickhouse" "clickhouse" {
    project                 = var.project_name
    cloud_name              = "google-europe-west1"
    plan                    = "startup-8"
    service_name            = "clickhouse-gcp-eu"
    maintenance_window_dow  = "monday"
    maintenance_window_time = "10:00:00"
  }

  resource "aiven_service_integration" "clickhouse_kafka_source" {
    project                  = var.project_name
    integration_type         = "clickhouse_kafka"
    source_service_name      = aiven_kafka.kafka.service_name
    destination_service_name = aiven_clickhouse.clickhouse.service_name
    clickhouse_kafka_user_config {
      tables {
	      name        = "edge_measurements_raw"
	      group_name  = "clickhouse-ingestion"
	      data_format = "JSONEachRow"
	      columns {
	        name = "sensor_id"
	        type = "UInt64"
	      }
	      columns {
	        name = "ts"
	        type = "DateTime64(6)"
	      }
	      columns {
	        name = "key"
	        type = "LowCardinality(String)"
	      }
	      columns {
	        name = "value"
	        type = "Float64"
	      }
	      topics {
	        name = aiven_kafka_topic.source.topic_name
	      }
      }
    }
  }
  
  resource "aiven_clickhouse_database" "measurements" {
    project                 = var.project_name
    service_name            = aiven_clickhouse.clickhouse.service_name
    name                    = "iot_measurements"
  }

Execute the Terraform files
'''''''''''''''''''''''''''

.. dropdown:: Expand to check out how to execute the Terraform files.

  1. Run the following command:

    .. code-block:: shell

       terraform init
  
  The ``init`` command performs initialization operations to prepare the working directory for use with Terraform. For this recipe, ``init`` automatically finds, downloads, and installs the necessary Aiven Terraform Provider plugins.

  2. Run the following command:

    .. code-block:: bash

       terraform plan -var-file=var-values.tfvars
  
  The ``plan`` command creates an execution plan and shows the resources to be created (or modified). This command doesn't actually create any resources but gives you a heads-up on what's going to happen next.

  3. If the output of ``terraform plan`` looks as expected, run the following command:

    .. code-block:: bash

       terraform apply -var-file=var-values.tfvars
  
  The ``terraform apply`` command creates (or modifies) your infrastructure resources.

Check out the results
---------------------

* Resource ``aiven_clickhouse`` creates an Aiven for ClickHouse service with the project name, the cloud name (provider, region, zone), the Aiven service plan, and the service name as specified in the ``services.tf`` file.
* Resource ``aiven_clickhouse_database`` creates a database that can be used to further transform the ingested data and perform analytics on it.
* Resource ``aiven_kafka`` creates an Aiven for Apache Kafka cluster.
* Resource ``aiven_kafka_topic`` creates Apache Kafka topic ``iot_measurements``.
* Resource ``aiven_service_integration`` creates the integration between the Aiven for Apache Kafka and the Aiven for ClickHouse service.

The service integration creates a database to insert the ingested data to. In this instance, the database name is ``service_kafka-gcp-eu`` (it depends on the Kafka service name) and the table name is ``edge_measurements_raw`` as specified in the code.

Learn more
----------

When you use this recipe, parameters and configurations will vary from those used in this article. For Aiven for Apache Kafka and Aiven for ClickHouse advanced parameters, a related blog, and instructions on how to get started with Aiven Terraform Provider, see `Set up your first Aiven Terraform project <https://docs.aiven.io/docs/tools/terraform/get-started.html>`_.

Follow up
---------

* You can `create databases and tables <https://docs.aiven.io/docs/products/clickhouse/howto/integrate-kafka.html#update-apache-kafka-integration-settings>`_ so that you can `read and store your data <https://docs.aiven.io/docs/products/clickhouse/howto/integrate-kafka.html#read-and-store-data>`_.
* You can also `create a materialized view <https://docs.aiven.io/docs/products/clickhouse/howto/materialized-views.html>`_ to store the Kafka® messages in Aiven for ClickHouse.
