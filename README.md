# aws_s3

Starting on Postgres version 11.1, AWS RDS added [support](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PostgreSQL.S3Import.html#USER_PostgreSQL.S3Import.FileFormats) for S3 import.
It allows to import data from S3 within Postgres using the function `aws_s3.table_import_from_s3`.

In order to support development either on RDS or locally, we implemented our own aws_s3 extension that will be similar to
the one provided in RDS. It was implemented in Python using the boto3 library.

## Installation
Make sure boto3 is installed using the default Python 2 installed on your computer.
On MacOS, this can be done as follows:

    sudo /usr/bin/easy_install boto3

Then install the extension:
    git clone git@github.com:chimpler/aws_s3
    
Make sure that `pg_config` can be run:
```
$ pg_config 

BINDIR = /Applications/Postgres.app/Contents/Versions/11/bin
DOCDIR = /Applications/Postgres.app/Contents/Versions/11/share/doc/postgresql
HTMLDIR = /Applications/Postgres.app/Contents/Versions/11/share/doc/postgresql
INCLUDEDIR = /Applications/Postgres.app/Contents/Versions/11/include
PKGINCLUDEDIR = /Applications/Postgres.app/Contents/Versions/11/include/postgresql
INCLUDEDIR-SERVER = /Applications/Postgres.app/Contents/Versions/11/include/postgresql/server
LIBDIR = /Applications/Postgres.app/Contents/Versions/11/lib
...
```

Then install aws_s3:

    make install
    
Then in Postgres:
```postgresql
psql> CREATE EXTENSION plpythonu;
psql> CREATE EXTENSION aws_s3;
``` 
    
## Using aws_s3

One can create a table that will import the data from S3. Let's create a table:
```postgresql
psql> CREATE TABLE animals (
    name TEXT,
    age INT
);
```

Let's suppose the following file is present in s3 at `s3://my-bucket/samples/myfile.csv`:
```
name,age
dog,12
cat,15
parrot,103
tortoise,205
```

The function `aws_s3.table_import_from_s3` has 2 signatures that can be used.

### Using s3_uri and aws_credentials objects

```postgresql
aws_s3.table_import_from_s3 (
   table_name text, 
   column_list text, 
   options text, 
   s3_info aws_commons._s3_uri_1,
   credentials aws_commons._aws_credentials_1
)
```

Using this signature, the s3_uri and aws_credentials objects will need to be created first:

Parameter | Description
----------|------------
table_name | the name of the table 
column_list | list of columns to copy
options | options passed to the COPY command in Postgres
s3_info | An aws_commons._s3_uri_1 composite type containing the bucket, file path and region information about the s3 object
credentials | An aws_commons._aws_credentials_1 composite type containing the access key, secret key, session token credentials

#### Example
psql> SELECT aws_commons.create_s3_uri(
   'my-bucket',
   'samples/myfile.csv',
   'us-east-1'
) AS s3_uri \gset

psql> \echo :s3_uri
(my-bucket,samples/myfile.csv,us-east-1)

psql> SELECT aws_commons.create_aws_credentials(
'<my_access_id>',
'<my_secret_key>',
'<session_token>'
) AS credentials \gset

psql> \echo :credentials
(<my_access_id>,<my_secret_key>,<session_token>)

psql> SELECT aws_s3.table_import_from_s3(
'animals',
'',
'(FORMAT CSV, DELIMITER '','', HEADER 1)',
:'s3_uri',
:'credentials'
)

 table_import_from_s3
----------------------
                    4
(1 row)

#### Using the function table_import_from_s3 with all the parameters

```postgresql
aws_s3.table_import_from_s3 (
   table_name text, 
   column_list text, 
   options text, 
   bucket text, 
   file_path text, 
   region text, 
   access_key text, 
   secret_key text, 
   session_token text 
) 
```

Parameter | Description
----------|------------
table_name | the name of the table 
column_list | list of columns to copy
options | options passed to the COPY command in Postgres
bucket | S3 bucket
file_path | S3 path to the file
region | S3 region (e.g., `us-east-1`)
s3_info | An aws_commons._s3_uri_1 composite type containing the bucket, file path and region information about the s3 object
credentials | An aws_commons._aws_credentials_1 composite type containing the access key, secret key, session token credentials


#### Example
```postgresql
psql> SELECT aws_s3.table_import_from_s3(
    'animals',
    '',
    '(FORMAT CSV, DELIMITER '','', HEADER 1)',
    'my-bucket',
    'samples/myfile.csv',
    '<my_access_id>',
    '<my_secret_key>',
    '<session_token>'
);

```
 table_import_from_s3
----------------------
                    4
(1 row)

### Configuration

One can tune some parameters such as:
* `aws_s3.buffer_chunk_size` (chunk size when reading data from S3). Default is 262,144 bytes.
* `aws_s3.buffer_lines` (number of lines to be imported in the table at each time). Default is 1,000 lines.

One can update these parameters at the session level or at database level.

At the session level:
```postgresql
SET aws_s3.buffer_chunk_size = 1000000;
SET aws_s3.buffer_num_lines = 1000;
```

At the database level:
```postgresql
ALTER DATABASE db SET aws_s3.buffer_chunk_size = 1000000;
ALTER DATABASE db SET aws_s3.buffer_num_lines = 1000;

-- Then reopen postgres
```

Then to check the values of the settings:
```postgresql
SELECT current_setting('aws_s3.buffer_chunk_size', true) AS chunk_size, current_setting('aws_s3.buffer_num_lines', true) as num_lines;

 chunk_size | num_lines
------------+-----------
 1000000    | 1000
(1 row)
```
