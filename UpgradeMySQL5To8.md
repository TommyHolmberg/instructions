# Upgrade MySQL 5.7 to 8.0

This instruction was made for MariaDB which roughly correspond to MySQL 5.7. I migrated it to MySQL 8.0.40.
The main difference is that authentication_string should be password in MySQL.

This is what worked for me. There's no guarantee that it will work for you, but I hope it does.

## Export from MariaDB 10.4.13 and MySQL 5.7

This instruction was made with MariaDB 10.4.13 but it should work for MySQL 5.7, but it is untested.

1.  **Export database**
    Export all tables except for mysql, which governs the users and priveleges.

    ```sh
    mysqldump -u root -p --all-databases --ignore-database=mysql --add-drop-table --routines --triggers --single-transaction > all_databases_without_mysql_backup.sql
    ```

1.  **Export users and priveleges by running this bat file.**

    > **Obs**: If you're using Mysql 5.7, the `authentication_string` should be replaced with `password`. (untested)

    ```bat
    @echo off
    REM Set MySQL credentials
    set MYSQL_USER=root

    @echo OBS: This only works if password is not set. It will not export root user or its privileges.

    @echo Exporting users
    mysql -u root --skip-column-names -A mysql -e "SELECT CONCAT('CREATE USER \'', user, '\'@\'', host, '\' IDENTIFIED WITH ', plugin, ' BY \'', authentication_string, '\';') FROM mysql.user WHERE user NOT IN ('mysql.session','mysql.sys','debian-sys-maint','root');" > create_users.sql

    @echo successfully exported create_users.sql

    @echo Exporting temp file grants.
    REM Step 1: Export SHOW GRANTS commands (with semicolon addition)
    mysql -u %MYSQL_USER% --skip-column-names -A -e "SELECT CONCAT('SHOW GRANTS FOR ''',user,'''@''',host,''';',';') FROM mysql.user WHERE user<>'' AND user<>'root'" > temp_grants.sql

    @echo Verify temp...
    REM Verify if temp_grants.sql was created successfully
    if not exist temp_grants.sql (
        @echo ERROR: Failed to generate SHOW GRANTS commands. Check MySQL credentials or permissions.
        pause
        exit /b
    )

    @echo Processing...
    REM Step 2: Process SHOW GRANTS commands to generate user_grants.sql
    if exist user_grants.sql del user_grants.sql

    for /f "usebackq delims=" %%i in ("temp_grants.sql") do (
        REM Execute SHOW GRANTS, remove IDENTIFIED BY, and add semicolon if needed
        for /f "delims=" %%j in ('mysql -u %MYSQL_USER% --skip-column-names -A -e "%%i" ^| findstr /v "IDENTIFIED BY"') do (
            echo %%j | findstr /r ";$" >nul || echo %%j; >> user_grants.sql
        )
    )

    @echo Clean up...
    REM Cleanup temporary file

    @echo Done! User grants have been exported to user_grants.sql, and create_users.sql has been created.
    pause

    ```

1.  **Verify user_grants.sql**
    Make sure the `user_grants.sql` lines looks like

    ```sql
    GRANT ALL PRIVILEGES ON `woo`.* TO `woo`@`%`;
    ```

1.  **Verify create_users.sql**
    Make sure the `create_users.sql` lines looks like

    ```sql
    CREATE USER 'woo'@'%' IDENTIFIED WITH mysql_native_password BY 'hashed_password';
    ```

    Be wary of the single quotes and that it says AS and not BY in front of the password.

## Importing

You may need to adjust the `my.ini` in the `mysql/bin` folder. Try with the default `my.ini`, and if the import fails try to set the max file size options. Read the documentation to see your options.

1. Import db

   ```sh
   mysql -u root -p < all_databases_without_mysql_backup.sql
   ```

2. Import the users and grants.

   ```sh
   mysql -u root -p mysql < /path/to/create_users.sql
   mysql -u root -p mysql < /path/to/user_grants.sql
   ```

3. Flush priveleges

   ```sh
   mysql -u root -p FLUSH PRIVILEGES;quit;
   ```
