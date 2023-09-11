# Run a MariaDB or MySQL container with a custom table
## Files needed
You need to create the following files:
- `.env`: to specify environment variables
- `docker-compose.yml`: the configuration file for the container to be launched
- `1-ddl.sql`: the SQL script to create a table in the database
- `2-ddl.sql`: the SQL script to create entries in the created table
### Environment variables
First, you need to set up the environment variables that you want to use/apply to your database.  
In a `.env` file, inform the following variables following the format `variable: 'value'`:
- `EDITOR`: the name of the image of the SQL editor you want to use: in the case of MariaDB, it is `mariadb` and, for MySQL, it is `mysql`
- `VERSION`: the version/tag of the image of the SQL editor you want to use; this includes version numbers, editions (etc); default set to `latest`
- `DB_NAME`: the name of the database to be created
- `DB_ROOT_PWD`: the password of the 'root' (super-user) account
- `DB_EXTRA_USER`: the name of an additional account for the database
- `DB_EXTRA_USER_PWD`: the password of this additional account
- `SCRIPT_PATH`: the (local) path toward SQL scripts to be executed at the first launch of the container
- `MAPPING_PORT`: the (local) port that redirects towards the entry port of the container (and, then, of the database)
### Docker compose file
Second, we need to set up the whole configuration of the container of the SQL editor.  
To do so, we need to fill in the `docker-compose.yml` file with the following code:
```
version: '3.1'
services:
  db:
    image: ${EDITOR:-mariadb}:${VERSION:-latest}
    restart: always
    environment:
      MYSQL_DATABASE: ${DB_NAME:-dbcountry}
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PWD:-azerty123!!}
      MYSQL_USER: ${DB_EXTRA_USER:-country}
      MYSQL_PASSWORD: ${DB_EXTRA_USER_PWD:-azerty1234!!}
    volumes:
    - ${SCRIPT_PATH:-./scripts}:/docker-entrypoint-initdb.d
    ports:
    - ${MAPPING_PORT:-3306}:3306
```
In this code:
- the `version` informs about the version of the YML language used for the docker-compose file; no need to change it
- we start a `service` (i.e., a container) which is named `db`
- the service is built from the image indicated after `image`
- the service will `always` `restart` if it is stopped
- `environment` variables are specified to ensure a proper work of the database:
    - a name do the database, indicated by `MYSQL_DATABASE`
    - the 'root' account password, indicated by `MYSQL_ROOT_PASSWORD`
    - an additional user is added, specifying:
        - the name of the user, indicated by `MYSQL_USER`
        - the password of the user, indicated by `MYSQL_PASSWORD`
- to let the container know about the scripts we will create, we need to "mount" a `volume` between a local folder and the `docker-entrypoint-initdb.d` folder in the container
- we can do a custom port-forwarding using `ports` between a local port (user-chosen) and the container port (depending on the SQL editor, 3306 here)
### Table creation script
Third, once the whole container *could* run, we can add some "within customization" for the databases/tables to be present when the container is launched.  
We need to fill in the `1-ddl.sql` file with the following simple SQL code:
```
-- USE dbname;
CREATE TABLE country (ID INT PRIMARY KEY NOT NULL AUTO_INCREMENT, Country VARCHAR(255), CountryCode VARCHAR(2), LangCode VARCHAR(2));
```
In this code:
- it is unncessary to use the SQL command `USE`: while connecting to the SQL editor, you would need to "enter" within a database but, here, the script is automatically launched within the user-created database
- then, we need to `CREATE` a `TABLE` named `country`, which is formed of the following fields:
    - an identifier for each entry:
        - named `ID`
        - which will be an `INT`eger number
        - it is the `PRIMARY KEY` of the table, so it is obviously a `NOT NULL` field
        - to avoid having to inform it each time we need to add an entry (and, then, looking at the bottom of the table just before), we set it to `AUTO_INCREMENT` so that numbers are automatically assigned
    - three fields of interest, all being strings (`VARCHAR`):
        - the name of the `Country`, of unlimited length
        - the code of the country itself (`CountryCode`) and of the language spoken there (`LangCode`), both being only two-lettered strings
### Entries creation script
Fourth, once the whole container *could* rune with a custom table added, we need to add some data in it!  
To do so, we need to fill in the `2-dml.sql` file with the following SQL code:
```
-- USE dbcountry;
INSERT INTO country (Country,CountryCode,LangCode) VALUES ('France','FR','fr');
INSERT INTO country (Country,CountryCode,LangCode) VALUES ('Allemagne','DE','de');
```
In this code:
- once again, normally (i.e., with a traditional SQL client), you would need to enter in the database named `dbname` here, but this is not needed for boot scripts like this one
- then, we add as many entries as we want; the format is always the same :
    - we want to `INSERT INTO` the table `country`
    - we specify which fields we aim to complete, here the `Country`, `CountryCode`, and `LangCode` are to be filled
    - we give the `VALUES` to fill in each field for this entry, which are, here, for instance, for the two entries:
        - `'France'` then `'Allemagne'`for `Country` field
        - `'FR'` then `'DE'` for `CountryCode` field
        - `'fr'` then `'de'` for `LangCode` field
## 'How-to' run the container
### Container launch
To launch the container using Docker, you need to open a terminal where all the previous files (`docker-compose.yml`, `.env`, `scripts` folder) are located.  
Then, just type in the terminal
```
docker compose up -d
```
This launches the container. To ensure it is running, just type in the terminal
```
docker container list
```
This lists all containers, including the one you just launched (normally!)
### Check the database, the table, and the entries
Once the container is running, it is open on a given port (depeding on the SQL editor) which has been "mapped" ("port-forwarded") on a port on the local machine. This is done in the `docker-compose.yml`.  
The container being running, we can "enter" it and access to the database. To do so, just type in the terminal
```
docker exec -it eng-db-1 mysql -p
```
Pay attention to two elements:
- `eng-db-1`: this is the `NAME` of the container, it might be different; run `docker compose list` to know it (it is the latest field on the right)
- `mysql`: this is the SQL editor, so be sure to replace this by the one you used.  

Once entered, the prompt asks for a password; this is the "root" account password, set in the `.env` file as `DB_ROOT_PWD`. Enter it.  
Once in the client, you can see all databases by typing the following
```
show databases;
```
If everything goes right, you should see the database you crated (following the value of `DB_NAME` in the `.env` file).  
You can then enter this database to see which tables it contains. You can do so by typing the following
```
use dbname;
show tables;
```
Here, pay again attention to `dbname`: you have to replace it by the `DB_NAME` you previously set in the `.env` file.  
Normally, the `show`command should show you the (single) table you created in this database, and that is named `country` (as set in the `1-ddl.sql` script).  
Now, we want to see if the entries were correctly assigned, i.e., if the `2-dml.sql` script has been executed properly. To do so, simply enter
```
select * from country;
```
And now, the entries done in the script should appear!
### Check the backstages
Finally, we can check what we should always look at after any execution, before even trying to access what we created: the logs!  
To do so, simply type
```
docker compose logs db
```
And then you can see everything that happened in background when launching the container.  
Scroll up a bit, and you may quickly find :
- lines indicating the creation of the table, of users and of passwords
- just below, two lines indicating that the two scripts `1-ddl.sql` and `2-dml.sql` have been executed.  

With these logs, you should normally already know whereas the container executed well or not. But it's always clearer to explore the object itself a little bit!
