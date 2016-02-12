#### pkleapostsql

- cp8
```
select current_user;
select tableowner from pg_tables where tablename= 'a';
select * from pg_stat_user_tables;
select * from pg_user;
```
revoke public schema privileges
```
revoke all privileges on schema public from public;
```

```
set session authorization user_name;
```

```
create rolename login noinherit
grant username to rolename
grant usage on schema dbname to rolename
```

==

```
revoke all ON DATABASE dbname FROM rolename;
```
basic connect authorization:
```
grant connect on database dbname TO rolename;
```

schema security level:
```
grant usage/create on schema schemanameto rolename;
```
table security level: (insert/update/delete/trigger/reference/truncate)
```
grant all on tablenameto role;
```
==
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
===

using crypt and gen_salt.
```
truncate account;
insert into account values(1,crypt('pass',gen_salt('md5'));
```
And this is very weird
```
select crypt('pass','123acv')='123acv' as auth from account;
```

==
enable timing
```
\timing
select crypt('pass',gen_salt('bf',8));
```

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

- cp9
```
\set ECHO_HIDDEN
\z account
```

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

==
kill other sessions
```
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = current_database() AND pid <> pg_backend_pid();
```

Note: current_database() = database() in mysql  

If one is not able to drop a certain database because clients try to connect to, can execute the following query
```
UPDATE pg_database set datallowconn = 'false' WHERE datname = 'database to drop';
```


postgresql.conf file config: ex
```
SELECT current_setting('work_mem');    -- must single quote!
show work_mem;
SELECT set_config('work_mem', '8 MB', true); -- means only current transaction is affected
```


- cp10
some config:
```
max_connections
work_mem
shared_buffers
```
hard disk setting:
```
fsync   -- setting forces each transaction to be written to the hard disk after each commit
checkpoint_segments
```

planner
```
effective_cache_size
random_page_cost
```

benchtest
```
pgbench -i test_database
pgbench -c 4 -T 50 test_database   //4 client, 50 second duration test
```
