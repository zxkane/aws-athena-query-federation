## Example Athena Connector

This module is meant to serve as a guided example for writing and deploying your own connector to enable Athena to query a custom source. The goal with this guided tutorial is to help you understand the development process and point out capabilities. Out of necessity some of the examples are rather contrived and make use of hard coded schemas to separate learning how to write a connector from learning how to interface with the target systems you will inevitably want to federate to. 

## What is a 'Connector'?

A 'Connector' is a piece of code that can translate between your target data source and Athena. Today this code is expected to run in an AWS Lambda function but in the future we hope to offer more options. You can think of a connector as an extension of Athena's query engine. Athena will delegate portions of the federated query plan to your connector. More specifically:

1. Your connector must provide a source of meta-data for Athena to get schema information about what databases, tables, and columns your connector has. This is done by building and deploying a lambda function that extends com.amazonaws.athena.connector.lambda.handlers.MetadataHandler in the athena-federation-sdk module. 
2. Your connector must provide a way for Athena to read the data stored in your tables. This is done by building and deploying a lambda function that extends com.amazonaws.athena.connector.lambda.handlers.RecordHandler in the athena-federation-sdk module. 

Alternatively, you can deploy a single Lambda function which combines the two above requirements by using com.amazonaws.athena.connector.lambda.handlers.CompositeHandler or com.amazonaws.athena.connector.lambda.handlers.UnifiedHandler. While breaking this into two separate Lambda functions allows you to independently control the cost and timeout of your Lambda functions, using a single Lambda function can be simpler and higher performance due to less cold start.

In the next section we take a closer look at the methods we must implement on the MetadataHandler and RecordHandler.

### MetadataHandler Details

Lets take a closer look at what is required for a MetadataHandler. Below we have the basic functions we need to implement when using the Amazon Athena Query Federation SDK's MetadataHandler to satisfy the boiler plate work of serialization and initialization. The abstract class we are extending takes care of all the Lambda interface bits and delegates on the discrete operations that are relevant to the task at hand, querying our new data source.

```java
public class MyMetadataHandler extends MetadataHandler
{
    /**
     * Used to get the list of schemas (aka databases) that this source contains.
     *
     * @param allocator Tool for creating and managing Apache Arrow Blocks.
     * @param request Provides details on who made the request and which Athena catalog they are querying.
     * @return A ListSchemasResponse which primarily contains a Set<String> of schema names and a catalog name
     * corresponding the Athena catalog that was queried.
     */
    @Override
    protected ListSchemasResponse doListSchemaNames(BlockAllocator allocator, ListSchemasRequest request) {}

    /**
     * Used to get the list of tables that this source contains.
     *
     * @param allocator Tool for creating and managing Apache Arrow Blocks.
     * @param request Provides details on who made the request and which Athena catalog and database they are querying.
     * @return A ListTablesResponse which primarily contains a List<TableName> enumerating the tables in this
     * catalog, database tuple. It also contains the catalog name corresponding the Athena catalog that was queried.
     */
    @Override
    protected ListTablesResponse doListTables(BlockAllocator allocator, ListTablesRequest request) {}

    /**
     * Used to get definition (field names, types, descriptions, etc...) of a Table.
     *
     * @param allocator Tool for creating and managing Apache Arrow Blocks.
     * @param request Provides details on who made the request and which Athena catalog, database, and table they are querying.
     * @return A GetTableResponse which primarily contains:
     *             1. An Apache Arrow Schema object describing the table's columns, types, and descriptions.
     *             2. A Set<String> of partition column names (or empty if the table isn't partitioned).
     */
    @Override
    protected GetTableResponse doGetTable(BlockAllocator allocator, GetTableRequest request) {}

    /**
     * Used to get the partitions that must be read from the request table in order to satisfy the requested predicate.
     *
     * @param blockWriter Used to write rows (partitions) into the Apache Arrow response.
     * @param request Provides details of the catalog, database, and table being queried as well as any filter predicate.
     * @note Partitions are partially opaque to Amazon Athena in that it only understands your partition columns and
     * how to filter out partitions that do not meet the query's constraints. Any additional columns you add to the
     * partition data are ignored by Athena but passed on to calls on GetSplits. Also note tat the BlockWriter handlers 
     * automatically constraining and filtering out values that don't satisfy the query's predicate. This is how we
     * we accomplish partition pruning. You can optionally retreive a ConstraintEvaluator from BlockWriter if you have
     * your own need to apply filtering in Lambda. Otherwise you can get the actual preducate from the request object
     * for pushing down into the source you are querying.
     */
    @Override
    public void getPartitions(BlockWriter blockWriter, GetTableLayoutRequest request) {}

    /**
     * Used to split-up the reads required to scan the requested batch of partition(s).
     *
     * @param allocator Tool for creating and managing Apache Arrow Blocks.
     * @param request Provides details of the catalog, database, table, andpartition(s) being queried as well as
     * any filter predicate.
     * @return A GetSplitsResponse which primarily contains:
     *             1. A Set<Split> which represent read operations Amazon Athena must perform by calling your read function.
     *             2. (Optional) A continuation token which allows you to paginate the generation of splits for large queries.
     * @note A Split is a mostly opaque object to Amazon Athena. Amazon Athena will use the optional SpillLocation and
     * optional EncryptionKey for pipelined reads but all properties you set on the Split are passed to your read
     * function to help you perform the read.
     */
    @Override
    protected GetSplitsResponse doGetSplits(BlockAllocator allocator, GetSplitsRequest request) {}
}
```

You can find example MetadataHandlers by looking at some of the connectors in the repository. athena-cloudwatch and athena-tpcds are fairly easy to follow along with.

Alternatively, if you wish to use AWS Glue DataCatalog as the authoritative (or supplemental) source of meta-data for your connector you can extend com.amazonaws.athena.connector.lambda.handlers.GlueMetadataHandler instead of com.amazonaws.athena.connector.lambda.handlers.MetadataHandler. GlueMetadataHandler comes with implementations for doListSchemas(...), doListTables(...), and doGetTable(...) leaving you to implemented only 2 methods. The Amazon Athena DocumentDB Connector in the athena-docdb module is an example of using GlueMetadataHandler.

### RecordHandler Details

Lets take a closer look at what is required for a RecordHandler. Below we have the basic functions we need to implement when using the Amazon Athena Query Federation SDK's MetadataHandler to satisfy the boiler plate work of serialization and initialization. The abstract class we are extending takes care of all the Lambda interface bits and delegates on the discrete operations that are relevant to the task at hand, querying our new data source.

```java
public class MyRecordHandler
        extends RecordHandler
{
    /**
     * Used to read the row data associated with the provided Split.
     * @param constraints A ConstraintEvaluator capable of applying constraints form the query that request this read.
     * @param spiller A BlockSpiller that should be used to write the row data associated with this Split.
     *                The BlockSpiller automatically handles chunking the response, encrypting, and spilling to S3.
     * @param recordsRequest Details of the read request, including:
     *                           1. The Split
     *                           2. The Catalog, Database, and Table the read request is for.
     *                           3. The filtering predicate (if any)
     *                           4. The columns required for projection.
     * @note Avoid writing >10 rows per-call to BlockSpiller.writeRow(...) because this will limit the BlockSpiller's
     *       ability to control Block size. The resulting increase in Block size may cause failures and reduced performance.
     */
    @Override
    protected void readWithConstraint(ConstraintEvaluator constraints, BlockSpiller spiller, ReadRecordsRequest recordsRequest){}
}
```

## How To Build & Deploy

You can use any IDE or even just comman line editor to write your connector. The below steps show you how to use an AWS Cloud9 IDE running on EC2 to get started but most of the steps are applicable to any linux based development machine.


### Step 1: Create your Cloud9 Instance

1. Open the AWS Console and navigate to the [Cloud9 Service or Click Here](https://console.aws.amazon.com/cloud9/)
2. Click 'Create Environment' and follow the steps to create a new instance using a new EC2 Instance (we recommend m4.large) running Amazon Linux.


### Step 2: Download The SDK + Connectors

1. At your Cloud9 terminal run `git clone https://github.com/awslabs/aws-athena-query-federation.git` to get a copy of the Amazon Athena Query Federation SDK, Connector Suite, and Example Connector.

### Step 3: Install Development Tools (Pre-Requisites)

1. This step may be optional if you are working on a development machine that already has Apache Maven, the AWS CLI, and the AWS SAM build tool for Serverless Applications. If not, you can run  the `./tools/prepare_dev_env.sh` script in the root of the github project you checked out.
2. To ensure your terminal can see the new tools we installed run `source ~/.profile` or open a fresh terminal. If you skip this step you will get errors later about the aws cli or sam build tool not being able to publish your connector.

Now run `mvn clean install -DskipTests=true` from the athena-federation-sdk directory within the github project you checked out earlier. We are skipping tests just to make the build faster. Normally you should let the tests as a matter of best practice.

### Step 4: Write The Code

1. Create an s3 bucket (in the same region you will be deploying the connector), that we can use for spill and to upload some sample data using the following command `aws s3 mb s3://BUCKET_NAME` but be sure to put your actual bucket name in the command and that you pick something that is unlikely to already exist.
2. (If using Cloud9) Navigate to the aws-athena-query-federation/athena-example folder on the left nav. This is the code you extracted back in Step 2.
3. Complete the TODOs in ExampleMetadataHandler by uncommenting the provided example code and providing missing code where indicated.
4. Complete the TODOs in ExampleRecordHandler by uncommenting the provided example code and providing missing code where indicated.
5. Run the following command from the aws-athena-query-federation/athena-example directory to ensure your connector is valid.  `mvn clean install`
6. Upload our sample data by running the following command from aws-athena-query-federation/athena-example directory. Be sure to replace BUCKET_NAME with the name of the bucket your created earlier.  `aws s3 cp ./sample_data.csv s3://BUCKET_NAME/2017/11/1/sample_data.csv`

### Step 5: Package and Deploy Your New Connector

We have two options for deploying our connector: directly to Lambda or via Serverless Application Repository. We'll do both below.

*Publish Your Connector To Serverless Application Repository*

Run `../tools/publish.sh S3_BUCKET_NAME athena-example` to publish the connector to your private AWS Serverless Application Repository. This will allow users with permission to do so, the ability to deploy instances of the connector via 1-Click form.

If the publish command gave you an error about the aws cli or sam tool not recognizing an argument, you likely forgot to source the new bash profile after
updating your development environment so run `source ~/.profile` and try again.

Then you can navigate to [Serverless Application Repository](https://console.aws.amazon.com/serverlessrepo/) to search for your application and deploy it before using it from Athena.

(Alternatively you can publish your connector directly to Lambda but for simplicity this tutorial uses Serverless Application Repository.)

### Step 6: Validate our Connector.

One of the most challenging aspects of integrating systems (in this case our connector and Athena) is testing how these two things will work together. Lambda will capture logging from out connector in Cloudwatch Logs but we've also tried to provide some tools to stream line detecting and correcting common semantic and logical issues with your custom connector. By running Athena's connector validation tool you can simulate how Athena will interact with your Lambda function and get access to diagnostic information that would normally only be available within Athena or require you to add extra diagnostics to your connector.

Run `../tools/validate_connector.sh --lambda-func <function_name> --schema schema1 --table table1 --constraints year=2017,month=11,day=1`
Be sure to replace lambda_func with the name you gave to your function/catalog when you deployed it via Serverless Application Repository.

If everything worked as expected you should see the script generate useful debugging info and end with:
```txt
2019-11-07 20:25:08 <> INFO  ConnectorValidator:==================================================
2019-11-07 20:25:08 <> INFO  ConnectorValidator:Successfully Passed Validation!
2019-11-07 20:25:08 <> INFO  ConnectorValidator:==================================================
```

### Step 7: Run a Query!

Ok, now we are ready to try running some queries using our new connector. Some good examples to try include (be sure to put in your actual database and table names):

`select * from "lambda:<function_name>".schema1.table1 where year=2017 and month=11 and day=1;`

`select transaction.completed, count(*) from "lambda:<function_name>".schema1.table1 where year=2017 and month=11 and day=1 group by transaction.completed;`

*note that the <function_name> corresponds to the name of your Lambda function.



