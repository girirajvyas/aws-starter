# aws-spring boot

# 1. Overview
Steps for setup and connect different services from spring boot application. For now endpoints are exposed to test and verify the same.

# 2. Setup
When you are connecting from code, the first thing you will need is to authenticate yourself and for that you have to generate the credentials. Once you have them then you have to add the dependency of `aws-java-sdk` in your pom.xml in case of maven projects so that you can call the AWS api.

## 2.1 Credentials - Access key and Secret Key
 - Do not use your root account to create security credentials
 - Login via IAM user -> IAM -> Access management -> Users -> select your name -> Security credentials tab
 - Access keys -> Create access keys (Single time download and next time you have to create a new one in case you forgot or missed)

## 2.2 Create Spring booot project 
We have taken example of spring boot here. 
 - Create spring boot app via:
   - [Spring initializer](https://start.spring.io/)
   - directly via STS

## 2.3 Maven Dependency
 - Amazon SDK makes it possible to interact with various Amazon services from our applications. 
 - In the pom.xml file add the Amazon SDK dependency.
 - You can get the latest version from: https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk
 - ```xml
     <!-- https://mvnrepository.com/artifact/com.amazonaws/aws-java-sdk -->
      <dependency>
          <groupId>com.amazonaws</groupId>
          <artifactId>aws-java-sdk</artifactId>
          <version>1.12.5</version>
      </dependency>
   ```

## 2.4 Common step: Credentials config
 - For anything you need to do with AWS, you are supposed to provide the aws credentials. Hence, Create config file that creates bean of amazon credentials that we will use with all the other configurations
 ```java
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import com.amazonaws.auth.AWSCredentials;
  import com.amazonaws.auth.AWSCredentialsProvider;
  import com.amazonaws.auth.AWSStaticCredentialsProvider;
  import com.amazonaws.auth.BasicAWSCredentials;

  @Configuration
  public class AwsCredentialsConfig {
  
   @Value("${amazon.aws.accesskey}")
   private String amazonAwsAccessKey;
   
   @Value("${amazon.aws.secretkey}")
   private String amazonAwsSecretKey;
   
   
   private AWSCredentials amazonAWSCredentials() {
       return new BasicAWSCredentials(amazonAwsAccessKey, amazonAwsSecretKey);
   }
   
   @Bean
   public AWSCredentialsProvider awsStaticCredentialsProvider() {
     return new AWSStaticCredentialsProvider(amazonAWSCredentials());
   }
  }
 ```

# 3. S3
Lets try to connect S2 from the code.

 ```java
 @Bean
  public AmazonS3 amazonS3() {
    return AmazonS3ClientBuilder 
        .standard() 
        .withCredentials(new AWSStaticCredentialsProvider(amazonAWSCredentials()))
        .withRegion(Regions.US_EAST_2) 
        .build();
  }
 ```
 
# 4. DynamoDB
For dynamodb if you see their is not official dependency that can be used,  
but we have a community version of spring data that helps with the task.

Example implementation is available at:
- [spring-data-dynamodb-examples](https://github.com/derjust/spring-data-dynamodb-examples)
  - [UserRepository.java](https://github.com/derjust/spring-data-dynamodb-examples/blob/master/src/main/java/com/github/derjust/spring_data_dynamodb_examples/simple/UserRepository.java)
 
 Maven dependency: https://mvnrepository.com/artifact/com.github.derjust/spring-data-dynamodb
 ```xml
   <dependency>
      <groupId>com.github.derjust</groupId>
      <artifactId>spring-data-dynamodb</artifactId>
      <version>5.1.0</version>
   </dependency>
  ```

create dynamoDB configuration  
@EnableDynamoDBRepositories will be imported from the dependency added above  
```java
@Configuration
@EnableDynamoDBRepositories(basePackageClasses = UserRepository.class)
public class AwsDynamoDbConfig {
  
  @Autowired
  public AWSCredentials amazonAWSCredentials;
  
  public AWSCredentialsProvider awsStaticCredentialsProvider() {
    return new AWSStaticCredentialsProvider(amazonAWSCredentials);
  }
  
  @Bean
  @Primary
  public DynamoDBMapperConfig dynamoDBMapperConfig() {
      return DynamoDBMapperConfig.DEFAULT;
  }

  @Bean
  public DynamoDBMapper dynamoDBMapper(AmazonDynamoDB amazonDynamoDB, DynamoDBMapperConfig config) {
      return new DynamoDBMapper(amazonDynamoDB, config);
  }
  
  @Bean
  public AmazonDynamoDB amazonDynamoDB() {
      return AmazonDynamoDBClientBuilder
          .standard()
          .withCredentials(awsStaticCredentialsProvider())
              .withRegion(Regions.US_EAST_1).build();
  }
}
```

Create Entity:  
All the annotations used in this entity are provided via `aws-java-sdk-dynamodb`
```java
package io.github.girirajvyas.aws.entity;

import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBAttribute;
import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBAutoGeneratedKey;
import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBHashKey;
import com.amazonaws.services.dynamodbv2.datamodeling.DynamoDBTable;

@DynamoDBTable(tableName = "User")
public class User {

  private String id;
  private String firstName;
  private String lastName;

  // Default constructor is required by AWS DynamoDB SDK
  public User() {}

  public User(String firstName, String lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }

  @DynamoDBHashKey
  @DynamoDBAutoGeneratedKey
  public String getId() {
    return id;
  }

  public void setId(String id) {
    this.id = id;
  }

  @DynamoDBAttribute
  public String getFirstName() {
    return firstName;
  }

  public void setFirstName(String firstName) {
    this.firstName = firstName;
  }

  @DynamoDBAttribute
  public String getLastName() {
    return lastName;
  }

  public void setLastName(String lastName) {
    this.lastName = lastName;
  }

  @Override
  public int hashCode() {
    final int prime = 31;
    int result = 1;
    result = prime * result + ((id == null) ? 0 : id.hashCode());
    return result;
  }

  @Override
  public boolean equals(Object obj) {
    if (this == obj)
      return true;
    if (obj == null)
      return false;
    if (getClass() != obj.getClass())
      return false;
    User other = (User) obj;
    if (id == null) {
      if (other.id != null)
        return false;
    } else if (!id.equals(other.id))
      return false;
    return true;
  }
}
```

## 4.2 Issues
Using 5.1.0 version of said dependency will have below issue:
 - https://github.com/derjust/spring-data-dynamodb/issues/233
 - ```console
     Parameter 1 of method dynamoDBMapper in io.github.girirajvyas.aws.config.AwsDynamoDbConfig required a single bean, but 2 were found:
	- dynamoDBMapperConfig: defined by method 'dynamoDBMapperConfig' in class path resource [io/github/girirajvyas/aws/config/AwsDynamoDbConfig.class]
	- dynamoDB-DynamoDBMapperConfig: defined in null
   ```
 - Marking dynamoDbMapperConfig @Primary made it work

## 4.3 References
 - https://github.com/derjust/spring-data-dynamodb
 - https://javatodev.com/spring-boot-dynamo-db-crud-tutorial/
 - https://www.baeldung.com/spring-data-dynamodb - Integration test
 - https://dzone.com/articles/getting-started-with-dynamodb-and-spring

# 5. Deploy your Spring boot app on EC2

## 5.0 Summary
 - Create user, other than root user (if already done ignore)
 - Create Spring boot App (with basic rest endpoint or any other existing app if you already have)
 - Create EC2 instance (steps can be defined here, make sure to enable tcp 8080 port in security configuration and have key pair generated if not have already)
 - Connect to this linux AMI via windows:
   - Putty 
   - WSL https://docs.microsoft.com/en-us/windows/wsl/about   

## 5.1 App creation
 - Considering you are following along from start. You already have a spring boot app working.
 - If not, you can create one via:
   - [Spring initializer](https://start.spring.io/)
   - directly via STS
 - Create dummy endpoint that we can access to verify its working

## 5.2 EC2 Instance creation
 - Login to console
 - [EC2 Dashboard]()
 - Select EC2 

## Connect to EC2
 
WSL:
Powershell: wsl --install
Ref: https://docs.microsoft.com/en-us/windows/wsl/install

Putty (For ssh):
 - Install: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html 
 - Convert .pem to .ppk file
 - user ec2-user@ipv4_address
   - Error: no authentication method found
   - Solution: Connection -> SSH -> AUTH -> browse .ppk FIle
 - Detailed video: https://www.youtube.com/watch?v=jv-dgOfFN4o

Winscp (For file transfer)
S3

After SSH:
Install Java:
 sudo yum install java-1.8.0
 sudo yum remove java-1.7.0-openjdk (In case you have to ininstall previous version)

From Unix:
scp -i /Users/username/downloads/
EC2Keypair.pem /Users/username/downloads/springbootproject/target/springbootproject-0.0.1-SNAPSHOT.jar ec2-user@ec2-11-111-11-111.compute-1.amazonaws.com:~

# 5.X Resources
 - https://cloudkatha.com/how-to-deploy-spring-boot-application-on-aws-ec2/
 - https://medium.com/@kgaurav23/deploying-hosting-spring-boot-applications-on-aws-ec2-7babc15a1ab6
 - https://www.javacodegeeks.com/2019/10/deploy-spring-boot-application-aws-ec2-instance.html
 - https://www.javacodegeeks.com/2019/09/how-to-launch-an-ec2-instance-in-aws.html
 - https://aws.amazon.com/blogs/devops/deploying-a-spring-boot-application-on-aws-using-aws-elastic-beanstalk/
 - SCP: https://en.wikipedia.org/wiki/Secure_copy_protocol

# Resources

