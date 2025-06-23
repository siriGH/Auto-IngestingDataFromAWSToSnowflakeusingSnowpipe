// Create database and schemas if not exists already
CREATE DATABASE IF NOT EXISTS MYDB;
CREATE SCHEMA IF NOT EXISTS MYDB.file_formats;
CREATE SCHEMA IF NOT EXISTS MYDB.external_stages;

//Alter your storage integration
alter storage integration s3_int
set STORAGE_ALLOWED_LOCATIONS = ('s3://awss3sirik/csv/', 's3://awss3sirik/json/');

  
// Create a file format object of csv type
CREATE OR REPLACE file format mydb.file_formats.csv_fileformat
    type = csv
    field_delimiter = ','
    skip_header = 1
    empty_field_as_null = TRUE;

// Create a stage object using storage integration
CREATE OR REPLACE stage mydb.external_stages.stage_aws_pipes
    URL = 's3://awss3sirik/csv/'
    STORAGE_INTEGRATION = s3_int
    FILE_FORMAT = mydb.file_formats.csv_fileformat;  	

// List the files in Stage
LIST @mydb.external_stages.stage_aws_pipes;

	
// Create a table to load these files
CREATE OR REPLACE TABLE mydb.public.emp_data 
(
  id INT,
  first_name STRING,
  last_name STRING,
  email STRING,
  location STRING,
  department STRING
);
  

// Create a schema to keep pipe objects
CREATE OR REPLACE SCHEMA mydb.pipes;

// Create a pipe
CREATE OR REPLACE pipe mydb.pipes.employee_pipe
AUTO_INGEST = TRUE
AS
COPY INTO mydb.public.emp_data
FROM @mydb.external_stages.stage_aws_pipes
pattern = '.*employee.*';

// Describe pipe to get ARN
DESC pipe employee_pipe;

// Get Notification channel ARN and update the same in event notifications SQS queue
	
// Upload the file and verify the data in the table after a minute
SELECT * FROM mydb.public.emp_data;
