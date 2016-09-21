## pkleapsql

###Chapter 8. PostgreSQL Security
####PostgreSQL default access privileges
```
select current_user;
select tableowner from pg_tables where tablename= 'a';
select * from pg_stat_user_tables;
select * from pg_user;
```

For mistrusted languages, such as plpythonu, the user cannot create functions unless he/she is a superuser. Ex
```
CREATE FUNCTION min (a integer, b integer)
  RETURNS integer
AS $$
  if a < b:
    return a
  else:
    return b
$$ LANGUAGE plpythonu;
// ERROR:  permission denied
```

revoke public schema privileges
```
revoke all privileges on schema public from public;
```
[stackoverflow similar issue](http://dba.stackexchange.com/questions/35316/why-is-a-new-user-allowed-to-create-a-table)




The database's role system can be used to partially implement this logic by delegating the authentication to another role after the connection is established
```
set session authorization user_name;
```

```
create rolename login noinherit
grant username to rolename
grant usage on schema dbname to rolename
```

####PostgreSQL security levels
#####Database security level
```
revoke all ON DATABASE dbname FROM public;
```
basic connect authorization:
```
grant connect on database dbname TO rolename;
```

#####schema security level
```
grant usage/create on schema schemanameto rolename;
```
#####table security level: (insert/update/delete/trigger/reference/truncate)
```
grant all on tablenameto role;
```
####Encrypting data
#####PostgreSQL role password encryption
```
create role rolename1 with login password '';
create role rolename1 with login password '';
select usename, passwd from pg_shadow where usename in ('rolename1','rolename2');
```
changename,password
```
alter role ke1 rename to ke2;
alter role ke1 with password '';
```


#####pgcrypto
######One-way encryption
md5 function, basic encryption
```
create table account(id int,password text);
insert into account values(1,md5('pass'));
```
using 'pre-select' before using that, must
```
create extension pgcrypto
```
```
select (md5('pass')=password) as auth from account;
```
using crypt and gen_salt.
```
truncate account;
insert into account values(1,crypt('pass',gen_salt('md5'));
```
And this is very weird
```
select crypt('pass','123acv')='123acv' as auth from account;
```

enable timing
```
\timing
select crypt('pass',gen_salt('bf',8));
```
######Two-way encryption
list all functions of encryption and decryption
```
\df encrypt
\df decrypt
```

using self key
```
select encrypt('pass','mykey','aes');
select decrypt('\x25d91d1827dc46348c7ad378eedb5aef','mykey','aes'); //get bytes
select convert_from('\1234','utf-8'); //convert to string
```

but this way is not safe. one can check at pg_stat_activity to get the key
We can use gpg.
```
gpg --gen-key
gpg --list-secret-key
/root/.gnupg/secring.gpg
------------------------
sec   2048R/2C914A6D 2015-06-25
uid                  Name <email>
ssb   2048R/56C8FA64 2015-06-25     //-a : dump into 
gpg -a --export 2C914A6D > /var/lib/postgresql/9.4/main/public.key
gpg -a --export-secret-key 56C8FA64 > /var/lib/postgresql/9.4/main/secret.key
chown postgres:postgres /var/lib/postgresql/9.4/main/public.key
chown postgres:postgres /var/lib/postgresql/9.4/main/secret.key
```

Test:
```
CREATE OR REPLACE FUNCTION encrypt (text) RETURNS bytea AS
$$
BEGIN
  RETURN pgp_pub_encrypt($1, dearmor(pg_read_file('public.key')));
END;
$$ Language plpgsql;

CREATE OR REPLACE FUNCTION decrypt (bytea) RETURNS text AS
$$
BEGIN
  RETURN  pgp_pub_decrypt($1, dearmor(pg_read_file('secret.key')));
END;
$$ Language plpgsql;
test=# SELECT decrypt(encrypt('hello'));
```

###Chapter 9. The PostgreSQL System Catalog and System Administration Functions
####The system catalog
```
\set ECHO_HIDDEN
\z account
```

####Getting the database cluster and client tools version
dump database:
first check whether version are consistent.
```
SELECT version (); //in psql
```
```
pg_dump --version //in cmd
```
dump command:
```
pg_dump -h localhost -U postgres -d ke > ke.sql
```

####Terminating and canceling user sessions
kill other sessions
```
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = current_database() AND pid <> pg_backend_pid();
```
pg_cancel_backend only cancels the current query and pg_terminate_backend kills the entire connection.  
Note: current_database() = database() in mysql     

If one is not able to drop a certain database because clients try to connect to, can execute the following query
```
UPDATE pg_database set datallowconn = 'false' WHERE datname = 'database to drop';
```

####Setting and getting database cluster settings
postgresql.conf file config: ex
```
SELECT current_setting('work_mem');    -- must single quote!
show work_mem;
SELECT set_config('work_mem', '8 MB', true); -- means only current transaction is affected
```


###Chapter 10. Optimizing Database Performance
####PostgreSQL configuration tuning
#####Memory settings
some config:
```
max_connections
work_mem
shared_buffers
```
#####hard disk setting:
```
fsync   -- setting forces each transaction to be written to the hard disk after each commit
checkpoint_segments
```

#####Planner-related settings
```
effective_cache_size
random_page_cost
```

#####Benchmarking 
```
pgbench -i test_database
pgbench -c 4 -T 50 test_database   //4 client, 50 second duration test
```
####Tuning PostgreSQL queries
In general, the ```NOT IN``` construct can sometimes cause performance issues because postgres cannot use indexes to evaluate the query.

#####The EXPLAIN command and execution plan
batch insert test: analyze, explain
```
CREATE TABLE test (id INT PRIMARY KEY,name TEXT NOT NULL);
INSERT INTO test SELECT n , md5 (random()::text) FROM generate_series (1, 10) AS foo(n);  // function name can be arbitrary
ANALYZE test;
EXPLAIN SELECT * FROM test;
```

calculate the cost:  
formula: no of relation pages * seq_page_cost + no of rows * cpu_tuple_cost
```
relpages*current_setting('seq_page_cost')::numeric + reltuples*current_setting('cpu_tuple_cost')::numeric as cost
FROM pg_class
WHERE relname='test';
```

To really execute and get the cost in real time, use EXPLAIN (ANALYZE).
```
EXPLAIN (ANALYZE) SELECT * FROM test WHERE id >= 10 and id < 20;
```

###Chapter 13. PostgreSQL JDBC
####Issuing a query and processing the results
#####Static statements
To insert, update, or delete rows, the executeUpdate method can be used:
```java
int rowCount = statement.executeUpdate("DELETE FROM account");
```
The returned integer value is the number of rows that were affected by the update or delete operation.
```java
String sql = "INSERT INTO account (first_name, last_name, email, password) VALUES ('John', 'Doe', '@.com', '123')";
statement.executeUpdate(sql, new String[]{"account_id"});
statement.executeUpdate(sql, new int[]{1}); //assuming that account_id is the first column
```
Obtain generated indexes.
```java
ResultSet newKeys = statement.getGeneratedKeys();
if(newKeys.next()){
  int newAccountID = newKeys.getInt("account_id");
}
```
#####Getting information about the table structure
Obtain metadata:
```java
ResultSet result = statement.executeQuery("select * from account");
ResultSetMetaData metaData = result.getMetaData();

int columnCount = metaData.getColumnCount();
for(int c=1;c<=columnCount; c++)
{
  String columnName = metaData.getColumnName(c);

  int columnType = metaData.getColumnType(c);
  String columnClass = metaData.getColumnClassName(c);
  String columnTypeName = metaData.getColumnTypeName(c);
}
```
