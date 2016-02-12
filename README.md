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

===========  

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
