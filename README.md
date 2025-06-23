# Auto-IngestingDataFromAWSToSnowflakeusingSnowpipe
end-to-end setup to automatically load files from AWS S3 into Snowflake tables using Snowpipe with Storage Integration and SQS notifications.

Created an end-to-end setup to automatically load files from AWS S3 into Snowflake tables using Snowpipe with Storage Integration and SQS notifications.
Below is the process I followed, along with learnings and troubleshooting experience.

Step 1: Set up Prerequisites
Create an SQS Queue
Go to Amazon SQS.
Create a new Standard Queue.
Copy the Queue ARN – you’ll need it later in Snowflake and S3.
Update Snowflake Pipe with the SQS ARN.
After creating the pipe in Snowflake, run:
DESC PIPE mydb. pipes.employee_pipe;
This returns a notificationChannelName (Snowflake’s internal ARN for SQS).
 Copy this ARN, and go back to S3.
 Set Up Event Notifications in S3
Go to the S3 bucket.
Click on "Properties".
Scroll down to "Event Notifications" → click Create event notification.
Give it a name (e.g., snowpipe-events).
Event type: Choose "PUT" (i.e., when a file is uploaded).
Prefix: Optional, e.g., csv/ if you're monitoring only a specific folder.
Send to: Choose SQS queue, and select the ARN you got from DESC PIPE.
Created an AWS S3 bucket and uploaded CSV files with consistent naming patterns (e.g., sp_employee_1.csv).
Updated my existing Storage Integration (s3_int) in Snowflake to allow access to the bucket paths.
Created a CSV File Format object in Snowflake to match the structure of incoming files (initially, delimited).

Step 2: Create External Stage
Created an external stage pointing to the S3 bucket using the storage integration and file format.

Step 3: Create Target Table
Created a target table (emp_data) to store the auto-loaded employee records.

Step 4: Create Snowpipe
Created a pipe object with AUTO_INGEST = TRUE, specifying a pattern to match employee-related files.
COPY INTO mydb. public.emp_data
FROM @mydb.external_stages.stage_aws_pipes
PATTERN = '.*employee.*';

Step 5: Enable Notifications
Retrieved the notification channel ARN from DESC PIPE employee_pipe and added it to the SQS queue in AWS.
Uploaded test files to S3 and verified auto-loading into Snowflake within a minute.

Troubleshooting Snowpipe:
At one point, the pipe didn’t ingest files as expected. Here's how I handled it:

Step 1: Check Pipe Status
SELECT SYSTEM$PIPE_STATUS('employee_pipe');
This shows whether the pipe is running and tracks the last file ingested.
executionState
Meaning: Shows whether the pipe is currently RUNNING, PAUSED, or SUSPENDED.

In My Case:
 executionState: "RUNNING" – which means the pipe is active and ready to ingest files automatically.
pendingFileCount
Meaning: Tells how many files are sitting in the queue (SQS) waiting to be processed by the pipe.

Use Case:
 If this number is greater than 0 and doesn’t go down over time, it may indicate an issue with the pipe, file format, or file content.
lastIngestedTimestamp
Meaning: Shows the timestamp when the most recent file was successfully loaded by the pipe.

Use Case:
 Useful to verify if the pipe is actively ingesting new data. If this doesn’t update after uploading new files, it may be time to check the SQS setup or file patterns.

Step 2: Review Copy History
SELECT * FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(...));
Found the FIRST_ERROR_MESSAGE — the delimiter didn’t match (file used |, but format was set to ,).
Step 3: Validate Pipe Load
SELECT * FROM TABLE(INFORMATION_SCHEMA.VALIDATE_PIPE_LOAD(...));
Helped confirm ingestion failures and pending files.

Fix:
Updated the file format to use the correct delimiter (|).
Reran COPY INTO manually for failed files to ingest them successfully.
Pipe Management Commands

Pause pipe:
 ALTER PIPE mydb.pipes.employee_pipe SET PIPE_EXECUTION_PAUSED = true;
Resume pipe:
 ALTER PIPE mydb.pipes.employee_pipe SET PIPE_EXECUTION_PAUSED = false;
View pipes:
 SHOW PIPES in schema mydb.pipes;

Learning:
Snowpipe is powerful for real-time file ingestion without scheduling.
Storage Integration + Notification Channel via SQS = secure, automated pipelines.
Troubleshooting through SYSTEM$PIPE_STATUS, COPY_HISTORY, and VALIDATE_PIPE_LOAD is critical.
It's important to maintain consistent file structure and naming patterns for smooth auto-ingestion.
If a file fails to load due to an error (e.g., incorrect delimiter, schema mismatch, or file format issue), Snowpipe does not retry it automatically even after the issue is fixed.
Important: After correcting the issue (e.g., updating file format or schema), you must manually load the failed file using Copy Commond. 
COPY INTO mydb.public.emp_data
FROM @mydb.external_stages.stage_aws_pipes
FILES = ('sp_employee_3.csv');
