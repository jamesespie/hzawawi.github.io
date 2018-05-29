---
layout: page
title: DynamoDB with c# and .Net Core.
permalink: /dynamodb-post-1/
---

### Objective
When started working with dynamodb database, I found the online documentation and online code samples were quite limited and bit confusing. So I decided to start writing few blogs around the things I learned along the way so it provides some helps for future developers.

Goal of this first blog, is to give a simple example to quickly setup dynamodb client and be able to do some basic operations with the dynamoDb.
Future blogs will be giving more deep guidelines about amazon dynamodb api and dynamodb core features.

### Dependencies

- #### AWSSDK.DynamoDBv2 package:
  .Net API to facilitate the interaction with aws dynamodb in order to execute different operations against the database
  such as (createTable, saveItem, retrieveItem,etc..)

- #### [Localstack](https://github.com/localstack/localstack): 
  framework that helps mocking different aws cloud applications, in our example we are going to rely on it to mock amazon dynamodb database.
  localstack really helpful to use when you want to develop a cloud application offline and reduce dependencies on the cloud infrastructure. 
  
### Code Sample
#### 1. DynamoDb client setup
 ```
 public class DynamoClient
 {
 	private readonly AmazonDynamoDBClient _amazonDynamoDBClient;
 	private readonly DynamoDBContext _context;

 	public DynamoClient()
 	{
 	    //localstack ignores secrets
 	    _amazonDynamoDBClient = new AmazonDynamoDBClient("awsAccessKeyId", "awsSecretAccessKey",
 	        new AmazonDynamoDBConfig
 	        {
 	            ServiceURL = "http://localhost:4569", //default localstack url
 	            UseHttp = true,
 	        });

 	    _context = new DynamoDBContext(_amazonDynamoDBClient, new DynamoDBContextConfig
 	    {
 	        TableNamePrefix = "test_"
 	    });
 	}
}
```

In order to create the client we need to pass to **AmazonDynamoDBClient** constructor the accessId and accessKey
and sessionToken in case you have MFA enabled for your account. For the sake of simplicity we are pointing to localstack mock dynamo mock service.


#### 2. Setup table and equivalent model class

```
[DynamoDBTable("student")]
public class Student : IEquatable<Student>
{
     [DynamoDBHashKey] 
     public int Id { get; set; }
     
     public string FirstName { get; set; }

     public string LastName { get; set; }

     public bool Equals(Student other)
     {
         if (ReferenceEquals(null, other)) return false;
         if (ReferenceEquals(this, other)) return true;
         return Id == other.Id && string.Equals(FirstName, other.FirstName) && string.Equals(LastName, other.LastName);
     }

     public override bool Equals(object obj)
     {
         if (ReferenceEquals(null, obj)) return false;
         if (ReferenceEquals(this, obj)) return true;
         if (obj.GetType() != this.GetType()) return false;
         return Equals((Student) obj);
     }

     public override int GetHashCode()
     {
         unchecked
         {
             var hashCode = Id;
             hashCode = (hashCode * 397) ^ (FirstName != null ? FirstName.GetHashCode() : 0);
             hashCode = (hashCode * 397) ^ (LastName != null ? LastName.GetHashCode() : 0);
             return hashCode;
         }
     }
}
```
In this example model, we are using two attributes:
- ***DynamoDBTable*** to map the dynamodb equivalent table
- ***DynamoDBHashKey*** to map the table hashkey

```
public async Task<CreateTableResponse> SetupAsync()
{
    var createTableRequest = new CreateTableRequest
    {
        TableName = "test_student",
        AttributeDefinitions = new List<AttributeDefinition>(),
        KeySchema = new List<KeySchemaElement>(),
        GlobalSecondaryIndexes = new List<GlobalSecondaryIndex>(),
        LocalSecondaryIndexes = new List<LocalSecondaryIndex>(),
        ProvisionedThroughput = new ProvisionedThroughput
        {
            ReadCapacityUnits = 1,
            WriteCapacityUnits = 1
        }
    };
    createTableRequest.KeySchema = new[]
    {
        new KeySchemaElement
        {
            AttributeName = "Id",
            KeyType = KeyType.HASH,
        },

    }.ToList();

    createTableRequest.AttributeDefinitions = new[]
    {
        new AttributeDefinition
        {
            AttributeName = "Id",
            AttributeType = ScalarAttributeType.N,
        }
    }.ToList();

    return await _amazonDynamoDBClient.CreateTableAsync(createTableRequest);
}
```

In this snippet, we are building the creation table request to construct the table, we have to specify all defined keys on the table. This seems bit of ceramony to do if you have quite few tables, we will show in the next blogs some solutions to make it easier to create tables associated to the models in a easier/faster maner for rapid developement.
For the sake of simplicity we are just going to stick with the raw api calls in this blog.


#### 3. Save/Query from the table

```
public async Task SaveOrUpdateStudent(Student student)
{
    await _context.SaveAsync(student);
}

public async Task<Student> GetStudentUsingHashKey(int id)
{
    return await _context.LoadAsync<Student>(id);
}

public async Task<Student> ScanForStudentUsingFirstName(string firstName)
{
    var search = _context.ScanAsync<Student>
    (
        new[]
        {
            new ScanCondition
            (
                nameof(Student.FirstName),
                ScanOperator.Equal,
                firstName
            )
        }
    );
    var result = await search.GetRemainingAsync();
    return result.FirstOrDefault();
}
```
- SaveOrUpdateStudent: saving a new entity is quite straight forward, all you want to do is to call `SaveAsync`, bare in mind Save will create/update the record, so in case the record already exists the method invocation will just override the data on the matched record
- GetStudentUsingHashKey: will retrieve the record back using our hashKey
- ScanForStudentUsingFirstName: will scan the whole table looking for a matching firstName

#### 4. Prevent overwriting existing records
```
public async Task SaveOnlyStudent(Student student)
{
    var identityEventTable = Table.LoadTable(_amazonDynamoDBClient, "test_student");

    var expression = new Expression
    {
        ExpressionAttributeNames = new Dictionary<string, string>
        {
            {"#key", nameof(student.Id)},
        },
        ExpressionAttributeValues =
        {
            {":key", student.Id},
        },
        ExpressionStatement = "attribute_not_exists(#key) OR #key <> :key",
    };

    var document = _context.ToDocument(student);

    await identityEventTable.PutItemAsync(document, new PutItemOperationConfig
    {
        ConditionalExpression = expression,
        ReturnValues = ReturnValues.None
    });
}
```
In order to prevent overriding the details of existing records, we need to leverage conditional expressions in dynamoDb.
So if we try to insert a record with a hashKey already presents in the table we will get *ConditionalCheckFailedException* being thrown

### Reference
You can find the above code snippets on [github](https://github.com/hzawawi/DynamoDbSamples)
 