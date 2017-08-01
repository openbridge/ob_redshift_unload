# Redshift Unload
This script is meant to simplify running a [Redshift `UNLOAD` command](http://docs.aws.amazon.com/redshift/latest/dg/r_UNLOAD.html). One of the gaps with the `UNLOAD` command is the fact it will not output a header row to the `UNLOADED` data. This script automatically retrieves and adds headers to the file before output all from the convenience of a simple Dcoker container.

## Requirements
Everything is built into the container, including [`psycopg2`](http://initd.org/psycopg/docs/install.html) and other OS and Python packages. You will need to have connection details to your Redshift host and the AWS credentials to write the `UNLOADED` data to an S3 bucket.

### Configuration file
The script requires a configuration file named ``config.json`` in the same directory. The config file is comprised of the following parameters set here to obtain database connection info, AWS credentials and any `UNLOAD` options you prefer to use.

A sample configuration file is below.

```json
{
    "db": {
        "host": "test.redshift.io",
        "port": "5439",
        "database": "db1",
        "user": "username",
        "password": "password"
    },
    "aws_access_key_id": "myawsaccesskeyid",
    "aws_secret_access_key": "myawssecretaccesskey",
    "unload_options": [
    	"ADDQUOTES",
    	"PARALLEL OFF",
    	"ALLOWOVERWRITE",
    	"GZIP",
    	"DELIMITER ','"
    ]
}
```

### Runtime parameters
The script can accept different runtime parameters:
* ``-t``: The table you wish to UNLOAD
* ``-f``: The S3 key at which the file will be placed
* ``-s`` (Optional): The file you wish to read a custom valid SQL WHERE clause from. This will be sanitized then inserted into the UNLOAD command.
* ``-d`` (Optional): The date column you wish to use to constrain the results  
* ``-d1`` (Optional): The desired start date to constrain the result set
* ``-d2`` (Optional): The desired end date to constrain the result set  
Note:  ``-s`` and ``-d`` are mutually exlusive and cannot be used together. If neither is used, the script will default to not specifying a WHERE clause and output the entire table.

## Examples
This command will unload the data in the table ``mytable`` which the ``datecol`` is between to the specified S3 location.

Running inside a container:
```python
python unload.py -t mytable -f s3://dest-bucket/foo/bar/output_file.csv -r datecol -r1 2017-01-01 -r2 2017-06-01
```
Running it via Docker command:
```python
docker run -it openbridge/redshift-unload python /unload.py -t mytable -f s3://dest-bucket/foo/bar/output_file.csv -r datecol -r1 2017-01-01 -r2 2017-06-01
```


### The `-s` Option
As mentioned previously, it is possible to supply your own WHERE clause to be used in the UNLOAD command. This is done via an external SQL file.

Here is an example. To use this functionality to UNLOAD only new users, you createD a SQL file called `new-users.sql` that contains ``WHERE is_new = true``. Pretty simple.

```python
python unload.py -t mytable -f s3://dest-bucket/foo/bar/output_file.csv -r datecol -r1 2017-01-01 -r2 2017-06-01 -s /new-users.sql
```

**NOTE**: The -s option will **only** work for the `WHERE` clause. If toy try other clauses it will fail. The script would need to be ehanced to do more.


### Security
It may be wise to create a read-only user on the database for the script to use. This will improve the security of the script by further protecting against SQL injection. For more information on how to do this, check the manual for your database type.
