* Create 3 schemas:

```
CREATE SCHEMA raw_data;
CREATE SCHEMA analytics;
CREATE SCHEMA adhoc;
```

* Create a group called "analytics_users"
  * SELECT access for tables under analytics and raw_data
  * All access for tables under adhoc

```
CREATE GROUP analytics_users;

GRANT USAGE ON SCHEMA analytics TO GROUP analytics_users;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO GROUP analytics_users;

GRANT USAGE ON SCHEMA raw_data TO GROUP analytics_users;
GRANT SELECT ON ALL TABLES IN SCHEMA raw_data TO GROUP analytics_users;

GRANT ALL ON SCHEMA adhoc to GROUP analytics_users;
GRANT ALL ON ALL TABLES IN SCHEMA adhoc TO GROUP analytics_users;
```

* Create a user called "bi_user" with a schema called "bi_user"

```
CREATE USER bi_user PASSWORD 'BiUserRocks!!1';
ALTER GROUP analytics_users ADD USER bi_user;  -- this can be optional
CREATE SCHEMA bi_user;
ALTER SCHEMA bi_user OWNER TO bi_user;
```
