# Shared database with ephemeral Release environments

[Release](https://releasehub.com/) allows you to quickly create on-demand, ephemeral and permanent environments that run in **your** AWS and GCP cloud accounts. Customers create environments by selecting a GitHub, BitBucket, or GitLab branch, creating a single YAML _Environment Template_, and either issuing a click, an API, or CLI command. Release supports mono and poly-repo, microservice and monolith, and - optionally - bring-your-own infrastructure as code to further customize your environment.

In this project, we will provide an illustrative example of how Release customers can create a shared database used by multiple ephemeral Release environments.

_If you'd like to learn more about ephmemeral, on-demand environments, please contact us at https://releasehub.com._

## Introduction to Ephemeral Databases

[Release Instant Datasets](https://docs.releasehub.com/reference-guide/instant-datasets) allow you to create and restore any existing snapshot from Amazon RDS or GCP CloudSQL into it's own ephemeral database instance and quickly attach it to an ephemeral environment with two lines of YAML like that shown below:

```yaml
# Release Application Template
datasets:
  - name_of_your_dataset
```

Release Instant Datasets are useful when your ephemeral test environments require all of the following: 

* _Real_ RDS or CloudSQL databases
* Identical copies of a recent production or santized database snapshot
* An isolated database instance with each environment
* Automatic teardown of the database when your ephemeral environment is terminated

By meeting these criteria, Release Instant Datasets allow customers to:

* improve test reliability by mirroring production as closely as possible and providing consistent test results
* avoid noisy neighbors that are inherent with shared infrastructure
* avoid false-negative test failures due to unexpected data or schema changes

However, if you want a real RDS or CloudSQL shared test database restored from a recent snapshot but you don't require a separate database instance for each ephemeral environment, using a shared database may be a better choice. A shared database not only simplifies your overall cloud footprint, but it may also improve resource utilization and reduce your infrastructure spend cost. 


## Shared databases for Release ephemeral environments

Today, each Release Instant Dataset is a single-use database that is torn down when the associated Release ephemeral environment is torn down. 

However, you can create your own shared database for your ephemeral environments using your cloud provider's existing services, infrastructure-as-code (e.g. Terraform), and your Release Environment Variables.

The diagram and subsequent discussion below provides an example of how you could build a shared Amaon RDS database for your ephemeral Release environments using AWS CloudFormation and AWS Secrets Manager:

![](aws-example-diagram.png)

See below for additional explanation: 

1. **Create a shared database by restoring an existing database's snapshot**. You should create the shared database in a VPC accessible by your Release-managed Kubernetes cluster. The easiest approach is to launch in the same Virtual Private Cloud (VPC) network as your cluster; if you want to use a different VPC, please refer to your cloud provider's documentation to establish private connectivity and DNS resolution between your VPCs.

2. **Make your shared database "discoverable"** by future ephemeral Release environments. There are many ways to accomplish this; for example, you could tag your databases and query for the database endpoint at runtime from your ephemeral application code, or you could instead write your shared database's DNS and credential info to a serverless key-value store like an Amazon DynamoDB table, AWS Parameter Store, AWS or GCP Secrets Manager, or 3rd-party secrets management services like [Doppler](https://dopler.com). 

3. **Provide a [Release Environment Variable](https://docs.releasehub.com/reference-guide/application-settings/default-environment-variables)** to tell your environment container(s) how they can find connection information for your shared database. For example, if you wrote your database endpoint, username, and password to a secret named `release/shared_db/customer_demo` in AWS Secrets Manager, you would place the following in your Release Environment Variables: 

    ```yaml
    # Release Environment Variables
    services:
        backendService:
            - key: SHARED_DB
              value: customer_demo
    ```

4. **Use your cloud provider's SDK or CLI within your container(s) to retrieve the shared database connection details**. Using the AWS CLI, our example might look like this:

    ```sh
    # AWS CLI example
    aws secretsmanager get-secret-value \
        --secret-id "release/shared_database/$RELEASE_SHARED_DB"
    ```


## Feedback

If we can improve this document, please share your feedback by opening an issue on this GitHub repository.