# Magento 2.3 Apache Docker Image

![alt text](images/magento2.png "Magento Logo")
![alt text](images/docker_logo.png "Docker Logo")
![alt text](images/ubuntu.png "Ubuntu Logo")

## Table of Contents

* [Software versions](#versions)
* [Install Docker on Ubuntu 18.04](#installation)
* [Creating a new Magento project](#newproject)
  * [Installation via Wizard](#via-wizard)
  * [Installation via Script](#via-script)
  * [Sample Data](#sample-data)
* [Set up an existing project](#existingproject)
* [Flip into developer mode](#devmode)
* Enable caches
  * [Magento](#cache)
  * [Redis](#redis)
  * [Varnish](#varnish)
  * [Opcache](#opcache)
* [Elasticsearch](#elasticsearch)
* [Xdebug](#Xdebug)
* [Profiler](#profiler)


## <a name="versions">Versions</a>

* Docker: 18.09.3
* Docker Compose: 1.23.0
* Magento: 2.3.1
* Apache: 2.4.25
* Php: 7.2.14
* Mariadb: 10.3.13
* Xdebug: 2.6.1
* Varnish: 4.1.10
* Redis: 5.0.4
* Elasticsearch: 6.7.2
* System: Debian GNU/Linux 9.6 (stretch)


### <a name="installation">Install Docker on Ubuntu 18.04</a>

Although we can install Docker and Docker Compose from the official Ubuntu repositories, they are several minor versions behind the latest release. So, we'll install Docker following the official documentation page (https://docs.docker.com/install/linux/docker-ce/ubuntu/).

#### Installation steps of Docker CE:

```
# Uninstall old versions
$ sudo apt-get remove docker docker-engine docker.io containerd runc

# Update the apt package index
$ sudo apt-get update

# Install packages to allow apt to use a repository over HTTPS:
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

# Add Docker’s official GPG key
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Use the following command to set up the stable repository. 
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

# Update the apt package index.
$ sudo apt-get update

# Install the latest version of Docker CE and containerd
$ sudo apt-get install docker-ce docker-ce-cli containerd.io

```

#### Verify the installation

Verify that Docker CE is installed correctly by running the hello-world image.
    
`$ sudo docker run hello-world`
    
This command downloads a test image and runs it in a container. When the container runs, it prints an informational message and exits.

Now Docker CE is installed and running.

#### [Optional] Configure Docker to start on boot

Enable the docker system service to start docker when the system boots: 

`$ sudo systemctl enable docker`

To disable this behavior, use disable instead.

`$ sudo systemctl disable docker`

#### [Optional] Docker without sudo

By default, you need to use sudo to run Docker commands.

If you don’t want to use sudo when you use the docker command, create a Unix group called _docker_ and add users to it.

**Warning:**
The docker group is root-equivalent; see [Docker Daemon Attack Surface details](https://docs.docker.com/engine/security/security/#/docker-daemon-attack-surface) and this blogpost on [Why we don't let non-root users run Docker in CentOS, Fedora, or RHEL](https://www.projectatomic.io/blog/2015/08/why-we-dont-let-non-root-users-run-docker-in-centos-fedora-or-rhel/).

To create the docker group and add your user:

1- Add the docker group if it doesn't already exist

`$ sudo groupadd docker`

2- Add your user to the docker group.

`$ sudo usermod -aG docker $USER`

Other optional configuration steps: (https://docs.docker.com/install/linux/linux-postinstall/)
 

### <a name="installation">Install Docker Compose</a>

In order to manage and execute docker-compose files we need to install the Docker Compose package.

The command below is slightly different than the one you'll find on the Releases page. By using the -o flag to specify the output file first rather than redirecting the output, this syntax avoids running into a permission denied error caused when using sudo.

Go to https://docs.docker.com/release-notes/docker-compose/ and look for the last release of docker-compose.

We'll check the current release and if necessary, update it in the command below:

```
$ sudo curl -L https://github.com/docker/compose/releases/download/1.23.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

Next we'll set the permissions:

`sudo chmod +x /usr/local/bin/docker-compose`

Then we'll verify that the installation was successful by checking the version:

`docker-compose --version`

This will print out the version we installed:

```
Output
docker-compose version 1.23.0, build a133471
```

## Creating a new Magento project

### <a name="newproject">Set up Apache-php and Mysql containers</a>

1. Create the following folder structure in our workspace folder.
  
   ```
   # Recommended path: ~/workspace/{project_name}/
   - data                  // mysql data, apache log files...
     |-- apache
     |-- elasticsearch
     |-- mysql
     |-- redis
   - docker                // Our magento2 docker project
   - src                   // Magento 2 source code
   ```
   ```bash
   $ mkdir -p data/apache data/elasticsearch data/mysql data/redis docker src
   ```
       
2. Clone the Docker project

    `$ git clone git@github.com:jcastellanos926/magento2-docker.git docker`
    
3. Create the .env file from .env.sample and configure it.

    ```bash
    $ cp .env.sample .env
    $ gedit .env
    ```
    
4. Go to the docker folder and execute: 

    ```bash
    $ docker-compose build
    $ docker-compose up -d
    ```
  
5. Install Magento 2 using composer.

    ```bash
    $ docker exec -it web bash
    $ composer create-project --repository=https://repo.magento.com/ magento/project-community-edition .
    $ composer install
    ```
        
    [More Info](https://devdocs.magento.com/guides/v2.3/install-gde/composer.html)   
    
6. Change apache user as the file owner of our files and set the right permissions following the official [Magento documentation](https://devdocs.magento.com/guides/v2.3/install-gde/prereq/file-system-perms.html)

    ```bash
    $ docker exec -it web bash
    $ chown -R www-data:www-data .
    $ find var generated vendor pub/static pub/media app/etc -type f -exec chmod u+w {} +
    $ find var generated vendor pub/static pub/media app/etc -type d -exec chmod u+w {} +
    $ chmod u+x bin/magento
    ```

7. Edit the /etc/hosts file in your local machine to point your custom domain names to localhost

    ```bash
    # Magento 2 project
    127.0.0.1 local-m2.com
    127.0.0.1 db
    127.0.0.1 redis
    ```

8. Copy .htaccess and .gitignore and files from the [magento2 github repository](https://github.com/magento/magento2) to your project.

    ```bash
    $ curl -O https://raw.githubusercontent.com/magento/magento2/2.3-develop/.htaccess
    $ curl -O https://raw.githubusercontent.com/magento/magento2/2.3-develop/.gitignore
    $ chown www-data:www-data .gitignore .htaccess
    $ chmod 644 .gitignore .htaccess
    ```

#### <a name="via-wizard">Installation via Magento Wizard</a>

1. Open your browser and go to your configured localhost domain
![alt text](images/landing-install.png "Installation 1")

2. Follow the installation wizard. Use the previously configured database credentials.
![alt text](images/database-credentials.png "Installation 2")

#### <a name="via-script">Installation via Magento Script</a>

Alternatively, we can install Magento using an installation script.

Open the file `web/bin/install` and put your database and Admin User credentials.

Restart the Docker containers to be sure that your changes were applied successfully.
```
$ docker-compose restart
```

To execute, run:
```bash
$ docker exec -it web install
```

#### <a name="sample-data">Sample data</a>

Look for your repo.magento.com credentials in the auth.json file and run:
```bash
$ docker exec -it web bash
$ magento sampledata:deploy
$ magento setup:upgrade
$ magento cache:clean
```

## Set up Magento 2 environment for existing project

### <a name="existingproject">Set up Apache-php and Mysql containers</a>

1. Follow 1-4 [Creating a new Magento steps](#newproject)

2. Clone your Magento project into the src folder

    ```bash
    $ docker exec -it web bash
    $ git clone {your-project-repo} .
    $ composer install
    ```

3. Change apache user as the file owner of our files and set the right permissions following the official [Magento documentation](https://devdocs.magento.com/guides/v2.3/install-gde/prereq/file-system-perms.html)

    ```bash
    $ docker exec -it web bash
    $ chown -R www-data:www-data .
    $ find var generated vendor pub/static pub/media app/etc -type f -exec chmod u+w {} +
    $ find var generated vendor pub/static pub/media app/etc -type d -exec chmod u+w {} +
    $ chmod u+x bin/magento
    ```

4. Import the database of your project

    `mysql -umagento -hdb -p magento < yourdatabase.sql`

5. Put the magento database credentials in your app/etc/env.php file. By default:

    ```php
    'db' => [
        'table_prefix' => '',
        'connection' => [
            'default' => [
                'host' => 'db',
                'dbname' => 'magento',
                'username' => 'magento',
                'password' => 'magento',
                'active' => '1'
            ]
        ]
    ]
    ```
    
### <a name="cache">Enable Magento Cache</a>

Add the following lines to app/etc/env.php

```
    'cache_types' => [
        'config' => 1,
        'layout' => 1,
        'block_html' => 1,
        'collections' => 1,
        'reflection' => 1,
        'db_ddl' => 1,
        'compiled_config' => 1,
        'eav' => 1,
        'customer_notification' => 1,
        'config_integration' => 1,
        'config_integration_api' => 1,
        'full_page' => 1,
        'config_webservice' => 1,
        'translate' => 1,
        'vertex' => 1
    ],
```

Execute:

````bash
$ magento config:set --scope=default --scope-code=0 system/full_page_cache/caching_application 2
````

### <a name="redis">Configure Redis</a>

The following commands will edit the app/etc/env.php file to enable the Redis cache.

Configure Redis default caching

```bash
$ bin/magento setup:config:set --cache-backend=redis --cache-backend-redis-server=redis --cache-backend-redis-db=0
```

Configure Redis page caching

```bash
$ bin/magento setup:config:set --page-cache=redis --page-cache-redis-server=redis --page-cache-redis-db=1
```

Configure Magento to use Redis for session storage

```bash
$ bin/magento setup:config:set --session-save=redis --session-save-redis-host=redis --session-save-redis-log-level=3 --session-save-redis-db=2
```

Clean cache storage manually to apply changes

```bash
$ rm -rf var/cache/*
```

Links and verification instructions:
* [Use Redis for the Magento page and default cache](https://devdocs.magento.com/guides/v2.3/config-guide/redis/redis-pg-cache.html)
* [Use Redis for session storage](https://devdocs.magento.com/guides/v2.3/config-guide/redis/redis-session.html)

 
### <a name="opcache">Enable OPCACHE</a>

Remove the semicolons (;) from opcache.ini file and reload apache

`service apache2 reload`

### <a name="varnish">Enable Varnish</a>

1. Add the following to docker-compose.yml
    ```
      varnish:
        build:
          context: ./varnish/
        container_name: varnish
        depends_on:
          - web
        volumes:
          - ./varnish/default.vcl:/etc/varnish/default.vcl
          - ./varnish/varnish:/etc/default/varnish
          - ./varnish/supervisord.conf:/etc/supervisor/conf.d/supervisord.conf
        ports:
          - "80:80"
          - "6082:6082"
        networks:
            - magento2
    ```

2. Change the port of the web docker service from `80:80` to `8080:8080`

Sources:

- [Configure Magento to use Varnish](https://devdocs.magento.com/guides/v2.3/config-guide/varnish/config-varnish-magento.html)
- [Final verification](https://devdocs.magento.com/guides/v2.3/config-guide/varnish/config-varnish-final.html)


## <a name="elasticsearch">ELASTICSEARCH</a>

1. In admin panel, go to Stores - Settings - Configuration - Catalog - Catalog Search

2. Configure:
    * Search Engine: Elasticsearch 6.0+
    * Server Hostname: elasticsearch
    ![alt text](images/elasticsearch-config.png "Elasticsearch Magento 2 backend configuration")

2. Click on _*Test connection*_ and Save Config.

3. Reindex catalog data.

    ````bash
    $ magento indexer:reindex
    $ magento cache:clean
    ````

4. Verify that Elasticsearch is working:

    ````bash
    $ docker exec -it elasticsearch bash
    $ curl http://localhost:9200/_cat/health?v&pretty
    ````

If correct, a message similar to the following will be displayed:

```bash
 epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks
 1519701563 03:19:23  elasticsearch green           1         1      0   0    0    0        0             0
```

Sources:
- [Install and configure Elasticsearch](https://devdocs.magento.com/guides/v2.3/config-guide/elasticsearch/es-overview.html)


## <a name="xdebug">XDEBUG FOR PHPSTORM (Optional)</a>

Enable or disable Xdebug

```bash
# Enable
$ docker exec -it web xdebug-enable

# Disable
$ docker exec -it web xdebug-disable
```

Links:
- [How to use Xdebug in PHPStorm](https://www.jetbrains.com/help/phpstorm/configuring-xdebug.html)
- [Docker Connection](https://stackoverflow.com/questions/43588317/phpstorm-xdebug-cant-find-file-when-connection-comes-from-docker-container)


## <a name="profiler">Enable profiler in Magento 2</a>

To enable or disable the html profiler:

````bash
 # enable
 $ magento bin/magento dev:profiler:enable html

 # disable
 $ magento bin/magento dev:profiler:disable

 # refresh varnish cache
 $ docker restart varnish
````

To use the SQL profiler add the following code to the app/etc/env.php

````php
'profiler' => [
    'class' => '\Magento\Framework\DB\Profiler',
    'enabled' => true,
],

# For example:

'db' => [
        'table_prefix' => 'mlx_',
        'connection' => [
            'default' => [
                'host' => 'db',
                'dbname' => 'magento',
                'username' => 'magento',
                'password' => 'magento',
                'model' => 'mysql4',
                'engine' => 'innodb',
                'initStatements' => 'SET NAMES utf8;',
                'active' => '1',
                'profiler' => [
                    'class' => '\Magento\Framework\DB\Profiler',
                    'enabled' => true,
                ],
            ]
        ]
    ],
````

Paste the following code in the index.php after `$bootstrap->run($app)`;

````php
    /** @var \Magento\Framework\App\ResourceConnection $res */
    $res = \Magento\Framework\App\ObjectManager::getInstance()->get('Magento\Framework\App\ResourceConnection');
    /** @var Magento\Framework\DB\Profiler $profiler */
    $profiler = $res->getConnection('read')->getProfiler();
    echo "<table cellpadding='0' cellspacing='0' border='1'>";
    echo "<tr>";
    echo "<th>Time <br/>[Total Time: ".$profiler->getTotalElapsedSecs()." secs]</th>";
    echo "<th>SQL [Total: ".$profiler->getTotalNumQueries()." queries]</th>";
    echo "<th>Query Params</th>";
    echo "</tr>";
    foreach ($profiler->getQueryProfiles() as $query) {
        /** @var Zend_Db_Profiler_Query $query*/
        echo '<tr>';
        echo '<td>', number_format(1000 * $query->getElapsedSecs(), 2), 'ms', '</td>';
        echo '<td>', $query->getQuery(), '</td>';
        echo '<td>', json_encode($query->getQueryParams()), '</td>';
        echo '</tr>';
    }
    echo "</table>";

````

Source:

- [Html Profiler](https://devdocs.magento.com/guides/v2.3/config-guide/bootstrap/mage-profiler.html)
- [DB Profiler](https://devdocs.magento.com/guides/v2.3/config-guide/db-profiler/db-profiler.html)

## <a name="devmode">Flip the system into developer mode</a>

```bash
$  php bin/magento deploy:mode:set developer
```
