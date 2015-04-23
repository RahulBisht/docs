# MySQL

1. Download and install MySQL Database (http://dev.mysql.com/downloads/mysql/)

2. Create a new database and a user capable of accessing this database with privileges for creating tables included (see MySQL documentation if you have questions about how to administrate databases and users).

3. Download the MySQL JDBC driver (http://dev.mysql.com/downloads/connector/j/) and place it in the classpath for your application server

4. Follow the instructions for your application server for creating a JNDI resource(s). Note that the driver does not go inside the war file(s). Rather it must go on the class path of the server.

6. Update the runtime properties to use the correct dialect for the MySQL. (see [[Database Configuration]]).

    ```ini
    blPU.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
    blSecurePU.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
    blCMSStorage.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect
    ```

> You may also want to check out the more detailed [[Switch To MySQL Tutorial]]

At a minimum, we also recommend using the following settings for proper UTF-8 configuration for MySQL in `my.cnf`:

```ini
[mysqld]
lower_case_table_names=1
character-set-server=utf8
collation-server=utf8_general_ci
```
