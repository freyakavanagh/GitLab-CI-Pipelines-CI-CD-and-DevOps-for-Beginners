# Java Project

# 1. Build Stage

Compilation: changes cource code(.java) into Java bytecode(.jar) that can be executed on the Java Virtual Machine.

1. Create a .gitlab-ci.yml file in the repo
2. Create the build stage
3. Create the build job
4. Use the gradle wrapper build command to compile the application into a .jar file

![](../Images2/build.png)

5. You can then browse the .jar file artifact

![](../Images2/browse.png)

# 2. Smoke Test

To test if the application responds at all.

1. Create the test stage and the smoke test job.

![](../Images2/smoketest.png)

- Download the curl command in the before_script a the image does not have it.
- execute the .jar file.
- sleep for 30 seconds to let it finish running.
- Test the health of the application, healthy gives an "UP" response

# 3. Deploy Manually using AWS Elastic Beanstalk

AWS Elastic Beanstalk: A way to deploy software, serverless architecture
  - Serverless computing is a cloud computing execution model in which the cloud provider allocates machine resources on demand, taking care of the servers on behalf of their customers.

1. Change region to Ireland
2. Search for AWS Elastic Beanstalk
3. Create new application
4. Web server environment (the api works over HTTP)
5. Name the environment

![](../Images2/beanstalk.png)

6. The domain will automaically generate
7. Platform: java
8. java 8 running on 64bit Amazon Linux

![](../Images2/beanstalk2.png)

9.  Application code: Sample Application (to see how the environment works)
10. Single instance (free)
11. Next

![](../Images2/beanstalk3.png)

12. Created and use new service role.
13. Duplicate tab and search for IAM
14. Roles
15. Create role
16. AWS Service
17. EC2
18. Next
19. Search for beanstalk
20. MulticontainerDocker
21. WebTier
22. WorkerTier
23. Next

![](../Images2/createrole.png)

24. Name
25. Create role

![](../Images2/namerole.png)

26. Refresh environment tab instance profile
27. Select EC2 Instance profile
28. Skip to review

![](../Images2/configure.png)

29. Submit
30. Wait for the environment to launch, then go to the Domain in 'environment overview'
31. You should see an elastic beanstalk website...

![](../Images2/beanstalkwebsite.png)

32. Upload and Deploy
33. Choose the .jar file
34. Deploy
35. Use PostMan to check the deployment...
36. Create a new environment with the URL created by elastic beanstalk

![](../Images2/postmanenvi.png)

37. Check the health, you should get "up" as a result

![](../Images2/postmanhealth.png)

38. Check if it can do a get request for all cars

![](../Images2/getcars.png)

# 4. Deploy Automatically to AWS from GitLab CI

Use 'AWS Command Line Interface' to deploy from your command line interface automatically.
Use Amazon s3 to store the .jar file.

1. Go to s3
2. Create bucket.
3. Name
4. Leave all options as they are.
5. Create bucket
6. Go to GitLab
7. Settings
8. CI/CD
9. Variables
10. Add Variable for bucket name

![](../Images2/s3bucketvariable.png)

11. Go to IAM
12. Users
13. Create user...

![](../Images2/userdetails2.png)

14. Add AmazonS3FullAccess
15. Next

![](../Images2/permissions.png)

16. Create User
17. Go to user and security credentials
18. Create Access Key
19. Command Line Interface

![](../Images2/Accesskey.png)

20. Download .csv
21. Create a CI/CD variable for access key and secret access key (using the correct aws names)

![](../Images2/unprotected.png)

1.   Go to pipeline script
2.   Create a deploy stage and a deploy job
3.   In the script configure the instance and upload the .jar file to the bucket

![](../Images2/pipelinedeploy.png)

25. The pipeline should work
26. Check the s3 bucket contains the .jar file

## Script Variables

27. You can make the .jar file name a script variable
28. Use the environment variable '' that has a unique id for the current pipeline.
29. Insert the environment variable within the .jar file name to help you keep track of the current version (with -v for version before it)

30. We now have to create an application version in the script
31. In the same line we can also provide the application name as a variable
32. Also the version label which is the version environment variable
33. And the source bundle of te bucket and the key which is the artifact name as variables
34. Then add another command that deploys the version we just created.
35. Also add a command in the build script to rename the.jar file to the variable name we created
36. We then need to add beanstalk full permissions to our user... (we can now delete the old permission if we want to as they are included in this one)

![](../Images2/adminaccess.png)

```
variables:
  ARTIFACT_NAME: cars-api-v$CI_PIPELINE_IID.jar
  APP_NAME: cars-api

stages:
  - build
  - test
  - deploy

build:
  stage: build
  image: openjdk:12-alpine
  script:     
    - ./gradlew build
    - mv ./build/libs/cars-api.jar ./build/libs/$ARTIFACT_NAME
  artifacts:
    paths:
      - ./build/libs/    

smoke test:
  stage: test 
  image: openjdk:12-alpine
  before_script:
    - apk --no-cache add curl
  script:
    - java -jar ./build/libs/$ARTIFACT_NAME &
    - sleep 30
    - curl http://localhost:5000/actuator/health | grep "UP"

deploy:
  stage: deploy
  image:
    name: amazon/aws-cli
    entrypoint: [""]
  script:
    - aws configure set region eu-west-1
    - aws s3 cp ./build/libs/$ARTIFACT_NAME s3://$S3_BUCKET/$ARTIFACT_NAME
    - aws elasticbeanstalk create-application-version --application-name $APP_NAME --version-label $CI_PIPELINE_IID --source-bundle S3Bucket=$S3_BUCKET,S3Key=$ARTIFACT_NAME
    - aws elasticbeanstalk update-environment --application-name $APP_NAME --environment-name "production-1" --version-label=$CI_PIPELINE_IID

```
