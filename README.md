# SaaSGlue File Watcher/Processor Data Pipeline
This repo includes code, configuration files and instructions for creating a data pipeline that watches for files in an AWS S3 bucket and performs word count analysis on each file. The analysis is performed with a clojure application compiled as a java jar file included in this repo named "word-count-0.1.0-SNAPSHOT-standalone.jar". 

All steps in the data pipeline are performed on serverless or ephemeral compute. The serverless compute tasks run in AWS Lambda in a SaaSGlue managed AWS account. The ephemeral compute tasks run on an EC2 instance which is created dynamically and terminated after all files have been processed.

## Prerequisites
1. SaaSGlue account - click [here](https://console.saasglue.com) to create an account
2. AWS account with S3 and EC2 permissions
    - You will need an S3 bucket containing one or more text files to analyze. The files can be prefixed however you prefer. e.g. "s3://my-bucket/file-watcher-pipeline/testfiles/".
    - You will need to define an IAM role providing read-only access to the AWS S3 bucket containing your input files. Permissions should include the following:

        ```
        {
            "Version": "2020-06-16",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "ec2:DescribeSubnets",
                        "ec2:DescribeSecurityGroups",
                        "ec2:DescribeInstances",
                        "ec2:DescribeImages",
                        "ec2:DescribeKeyPairs",
                        "ec2:DescribeVpcs",
                        "ec2:CreateSecurityGroup",
                        "ec2:AuthorizeSecurityGroupIngress",
                        "ec2:CreateKeyPair"
                    ],
                    "Resource": "*"
                },
                {
                    "Effect": "Allow",
                    "Action": "ec2:RunInstances",
                    "Resource": "*"
                }
            ]
        }
        ```

        and

        ```
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "s3:Get*",
                        "s3:List*"
                    ],
                    "Resource": "*"
                }
            ]
        }
        ``` 

## Install
1. Clone the file-watcher-data-pipeline repo
    ```
    $ git clone --depth 1 https://github.com/saascipes/file-watcher-data-pipeline.git
    ```
2. Configure SaaSGlue
    1. Log in to the SaaSGlue web console
        - Log in to the SaasGlue web [console](https://console.saasglue.com) - if you don't have an account yet, sign up for a free account
        - Click your name in the upper right corner and click "settings" - note your "Team ID" - you'll need this later
    2. Create SaaSGlue access keys
        1. Create Agent access key
            - Log in to the SaasGlue web [console](https://console.saasglue.com)
            - Click your login name in the upper right hand corner and click "Access Keys"
            - Click the "Agent Access Keys" tab
            - Click "Create Agent Access Key"
            - Enter a description, e.g. "default"
            - Click "Create Access Key"
            - Copy the access key secret
            - Click the "I have copied the secret" button
            - Copy the access key id from the grid
        2. Create API access key
            - Log in to the [SaasGlue web console](https://console.saasglue.com)
            - Click your login name in the upper right hand corner and click "Access Keys"
            - Click the "User Access Keys" tab
            - Click "Create User Access Key"
            - Enter a description, e.g. "Twitter data pipeline"
            - Click "Select None"
            - Click the checkboxes next to 
                - JOB_CREATE
                - AGENT_WRITE
                - AGENT_READ"
                - AGENT_STUB_DOWNLOAD
            - Click "Create Access Key"
            - Copy the access key secret
            - Click the "I have copied the secret" button
            - Copy the access key id from the grid

    3. Import SaaSGlue jobs
        - Click "Designer" in the menu bar
        - Click "Import Jobs"
        - Click "Choose File"
        - Select the "file-watcher-data-pipeline.sgj" file in the file-watcher-data-pipeline root folder and click "Open"
        - This will import three jobs:
            - File Watcher - checks for files in an AWS S3 location and runs the "File Analyzer" job for each file it finds. This job consists of four tasks.
                - Tasks
                    - Check for new files - this task runs in AWS Lambda. It runs a NodeJS script which gets a list of files in a particular AWS S3 bucket/prefix and passes the list of files to subsequent tasks in the job as a runtime variable named "files".
                    - Launch analyzer host - also runs in AWS Lambda. It runs a Python script that creates one or more EC2 instances to run the File Analyzer jobs. You can control the number and type of EC2 instances created by modifying the "NumInstances" and "InstanceType" runtime variables for the job. The job runtime variables are accessible in the web console in the Designer view, Runtime Variables tab.
                    - API Login - also runs in AWS Lambda. It runs a Python script that logs into the SaaSGlue API and gets a security token which is passed as a runtime variable to the next task in the job.
                    - Create analyze jobs - also runs in AWS Lambda. It runs a NodeJS script which creates an instance of the "File Analzyer" job for each file found in the "Check for new files" task by calling the SaaSGlue API. 
                - Note: The workflow for this job uses dynamic routing from the "Check for new files" task to the "Launch analyzer host" task so that if no files are found the job will end after the "Check for new files" task.
            - File Analyzer - performs word count analysis on a text file. This job consists of a single task named "Analyze file" which has three steps.
                - Steps
                    - download file - downloads the file from AWS S3 to the runtime environment.
                    - analyze file - runs a Clojure program compiled to a java jar file which generates a count of the number of instances of each word in the file and writes the results to std out.
                - Note: this job does not delete or move the input files, so if you want this behavior you would need to modify the script for the "download file" step to do that.
            - Stop Agent and Terminate EC2 - this job will terminate the EC2 intance that was created in the File Watcher job after it becomes inactive. It consists of three tasks.
                - Tasks
                    - API Login - runs the same script as the corresponding task in the File Watcher job
                    - Get EC2 inst id and region - this task runs on the agent host machine that will be terminated. It runs a shell script that gets the AWS instance id and region of the host EC2 instance and dynamically creates runtime variables that will be consumed in the next task.
                    - Terminate agent - runs in AWS Lambda. It runs a NodeJS script that terminates an EC2 instance using the instance id and region runtime variables created in the previous task.

        - After completing step 2 you will be able to view the imported jobs, tasks, steps and scripts in the SaaSGlue web [console](https://console.saasglue.com) in the Designer view.
    4. Get the File Analyzer job id
        - From the Designer view, click the "File Analyzer" job
        - Click the "Settings" tab
        - Copy the text next to the "Id" label, e.g. 61318a332d748c0bdb7178ac
    5. Get the Stop Agent and Terminate EC2 job id
        - From the Designer view, click the "Stop Agent and Terminate EC2" job
        - Click the "Settings" tab
        - Copy the text next to the "Id" label, e.g. 61318a332d748c0bdb7178ac
    6. Configure runtime variables - the scripts that run in the imported jobs use runtime variables that are defined in the jobs. Follow these steps to set the runtime variables for each job.
        - Log in to the SaaSGlue web [console](https://console.saasglue.com)
        - Click "Designer" in the menu bar
        - To set the runtime variables for a job, click the job name and then select the "Runtime Variables" tab
        - Then enter the job runtime variable key/value pairs - if there is an existing runtime variable with the given key, click "unmask" and enter the new value in the input box and hit "enter" - otherwise enter the key/value pair in the input boxes at the bottom of the grid and then click "Add Runtime Variable"
        - Now set the runtime variables for the "File Watcher" job as follows:

            ```
            instanceNameTag = the name tag to assign to the EC2 instance that will process the files
            NumInstances = 1
            ImageId = ami-0fcfc80303c1faf87
            sgAPIAccessKeySecret = [your api access key secret from step 2.2.2]
            sgAPIAccessKeyId = [your api access key id from step 2.2.2]
            s3Prefix = [the AWS S3 prefix where your input file(s) are located, e.g. file-watcher-demo]
            s3Bucket = [the AWS S3 bucket where your input file(s) are located, e.g. sg-files-share]
            jobDefId = [the File Analyzer job id from step 2.4]
            workingdir = /home/ec2-user
            tags = {"task":"file_analyzer"}
            maxAgentTasks = 2
            inactivePeriodWaitTime = 60000
            VpcId = [your AWS vpc id]
            SubnetId = [the id of the AWS subnet where you want to deploy your EC2 instance that will do the file analysis]
            SecurityGroupIds = [the security group(s) to apply to your EC2 instance, e.g. ['sg-0cc6a8d7e42edd802'] (including brackets!))
            KeyName = [the AWS key pair name to apply to your EC2 instance]
            InstanceType = [the instance type to use for your EC2 instance, e.g. t3.small]
            InactiveAgentJobId = [the Stop Agent and Terminate EC2 job id from step 2.5]
            IAMRole = [the name of your AWS IAM Role (detailed requirements in Prerequisites section)]
            agentAccessKeySecret = [your agent access key secret from step 2.2.1]
            agentAccessKeyId = [your agent access key id from step 2.2.1]
            ```

    7. Upload word count jar file
        - Click "Artifacts" in the menu bar
        - Click "Upload New Artifacts"
        - Click "Choose Files"
        - Browse to the location of "word-count-0.1.0-SNAPSHOT-standalone.jar", click on the file and then click "Open"
        - Click "Upload Artifacts" - depending on your internet speed it may take up to a minute to upload the file
        - Click "Close"
    8. Attach jar file to "File Analyzer" job, "Analyze file" task
        - Click "Designer" in the menu bar
        - Click the "File Analyzer" job
        - Click the "Analyze file" task in the side bar
        - Scroll down to the "Add Artifact(s)" button and click on it
        - Enter "word-count" in the input box with the hint "Artifact name"
        - Click "word-count-0.1.0-SNAPSHOT-standalone.jar" in the selection box
        - Click "Add Artifact(s)"
        - When the "Analyze file" task runs, the jar file will be downloaded to the runtime environment to run the file analysis
## Usage
To start the data pipeline that analyzes the input files:
    - From the Designer view in the SaaSGlue web console, click the "File Watcher" job link
    - Click the "Run" tab
    - Click "Run Job"
    - Click "Running job link

You can view the details of each task in the job by clicking on the task name hyperlink. As each step in each task progresses, you will see the stdout tail updating. To view the running script, click the "script" hyperlink. To view the full stdout, click the "stdout" link and similarly for "stderr".

When the File Watcher job tasks are complete, click "Monitor" in the menu bar. You should see a job instance for each of your input files, e.g. "Analyze file - [s3 file path]". You can click the "Monitor" link for each job to monitor the progress. It will take a minute or so for the EC2 instance created in the File Watcher job to come up and for the SaaSGlue Agent to start up. Click "Agents" in the menu bar to watch for the new Agent to connect. This will take a minute or so. 

When the new Agent appears in the list, click "Monitor" from the menu bar and click on the "Monitor" link next to the running File Analyzer job. When the task completes you should see a list of the words in the input file followed by the number of occurrences of the word. E.g.

    [fully 1]
    [worlds 1]
    [high 5]
    [this 4]
    [of 12]

After all File Analyzer jobs are complete and after a waiting period of 60 seconds, the "Stop Agent and Terminate EC2" job will run automatically. It will be named "Inactive agent job - [EC2 instance name]" in the job Monitor. Click on the "Monitor" link next to the job instance to view the running details. After this job is complete, the EC2 instance created by the File Watcher job will be terminated.

