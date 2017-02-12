# AWS Elastic Beanstalk Wordpress Deployment

## Introduction

Have you ever been scared on clicking on that WordPress or plugins update button ?
Have you ever wanted to have a staging environement for the validation of your latest development by your customers and to update the production with one push upon validation ?
Have you ever wanted to have automated test to ensure proper site function before deployments ?

The goal of this project is to have an WordPress environment that let you develop, test, package and deploy an immutable WordPress on different environments like staging and production.

It is based on docker-compose for the local development, [continuousphp](https://continuousphp.com) for building, testing and deploying on Elastic BeanStalk Infrastructure environments.

So let's start! 

## Setting your Development environment

### Dependencies managment with composer

Install Wordpress and plugins using composer like:

```
composer.phar install
```

Create the symlinks to enable plugins and themes in development mode:

```
./vendor/bin/phing wp-symlink-plugins
./vendor/bin/phing wp-symlink-themes
```

Most of the WP plugins can be found on [wordpress packagist](https://wpackagist.org/), add them to your composer.json like:

```
    "require": {
        "php": ">=7.0",
        "johnpbloch/wordpress": ">=4.7.2",
        "composer/installers": "~1.0",
        "wpackagist-plugin/contact-form-7": "4.6.1",
        "wpackagist-plugin/business-profile": "1.1.1",
        "robmorgan/phinx": "~0.6.0",
        "wp-cli/wp-cli": "^1.0",
        "phing/phing": "~2.14"
    },
```

### Set the Phing environment variables

To set your local development environment copy the build.local.properties.sample to build.local.properties.

```
cp build.local.properties.sample build.local.properties
``` 

### Configure Wordpress

```
cp wp/wp-config-sample.php wp/wp-config.php 
```
Edit the wp/wp-config.php file and set WP configuration variables using $_SERVER environment variables as below, we do this to be able to provision the WP configuration for the differents environments like develop, staging and production.

```
/** The name of the database for WordPress */
define('DB_NAME',     $_SERVER["MYSQL_ADDON_DB"]);

/** MySQL database username */
define('DB_USER',     $_SERVER["MYSQL_ADDON_USER"]);

/** MySQL database password */
define('DB_PASSWORD', $_SERVER["MYSQL_ADDON_PASSWORD"]);

/** MySQL hostname */
define('DB_HOST',     $_SERVER["MYSQL_ADDON_HOST"]);

/** Database Charset to use in creating database tables. */
define('DB_CHARSET', 'utf8');

/** The Database Collate type. Don't change this if in doubt. */
define('DB_COLLATE', '');
```

And Add:

```
define('AUTH_KEY',         $_SERVER["AUTH_KEY"]);
define('SECURE_AUTH_KEY',  $_SERVER["SECURE_AUTH_KEY"]);
define('LOGGED_IN_KEY',    $_SERVER["LOGGED_IN_KEY"]);
define('NONCE_KEY',        $_SERVER["NONCE_KEY"]);
define('AUTH_SALT',        $_SERVER["AUTH_SALT"]);
define('SECURE_AUTH_SALT', $_SERVER["SECURE_AUTH_SALT"]);
define('LOGGED_IN_SALT',   $_SERVER["LOGGED_IN_SALT"]);
define('NONCE_SALT',       $_SERVER["NONCE_SALT"]);
```

Once finished, copy the wp/wp-config.php in the wp-root/ folder, we will need it for the deployment provisioning. 

Let's xport the environment variables for our development environents:

```
export MYSQL_ADDON_HOST=192.168.99.100
export MYSQL_ADDON_DB=wordpress
export MYSQL_ADDON_PASSWORD=password
export MYSQL_ADDON_USER=root
export S3_BACKUP_URL=
export S3_MEDIA_URL=
export ENVIRONMENT=develop
export AUTH_KEY=""
export SECURE_AUTH_KEY=""
export LOGGED_IN_KEY=""
export NONCE_KEY=""
export AUTH_SALT=""
export SECURE_AUTH_SALT=""
export LOGGED_IN_SALT=""
export NONCE_SALT=""
```
 
#### Initialize your Database

Now let's initialize our WP database.

```
./vendor/bin/phing reset-db
```

#### WordPress Base Install

Open in your browser http://192.168.99.100/wp-admin/install.php to install your WordPress.
Complete the information and click **Install WordPress**

Your WordPress is now ready to customized.

### Configure your development environment

```
./vendor/bin/phing setup-dev
```

#### Plugin Install

We are going to add a few plugins. 

Updraftplus to backup our install on S3, for this we add the "wpackagist-plugin/updraftplus" dependency in our composer.json file.
The w3-total-cache Search Engine and Performance Optimization plugin and wordpress-https to enable SSL certificate for our site.

```
    "require": {
        "php": ">=7.0",
        "johnpbloch/wordpress": ">=4.7.2",
        "composer/installers": "~1.0",
        "wpackagist-plugin/contact-form-7": "4.6.1",
        "wpackagist-plugin/business-profile": "1.1.1",
        "wpackagist-plugin/updraftplus": "1.12.32",
        "wpackagist-plugin/w3-total-cache": "0.9.5.1",
        "wpackagist-plugin/wordpress-https": "3.3.6",
        "robmorgan/phinx": "~0.6.0",
        "wp-cli/wp-cli": "^1.0",
        "phing/phing": "~2.14"
    },
```

Edit the build.properties and add our new plugins into the wp.plugins variables.

```
wp.plugins=business-profile,contact-form-7,updraftplus,w3-total-cache,wordpress-https
```

Do the same to your build.local.properties.

Run a composer update to install the plugin.

```
composer.phar update
```

Activate the plugins:

```
./vendor/bin/phing wp-plugins-activate
```

Open in your browser http://192.168.99.100/wp-admin/plugins.php the updraftplus plugin is installed and activated. Note that the password for the admin user is password, we have updated it so we can run automated test.

Notice all the plugin are installed and activated, but wait a minute, we W3 Total Cache that require an update, let's edit our composer.json and bump the version.

Get the latest version at [WordPress Packagist](https://wpackagist.org/search?q=w3-total-cache&type=any&search=)

``` 
        "wpackagist-plugin/w3-total-cache": "0.9.5.2",
```

Run composer update and refresh the plugin page, our plugin is up to date!

```
composer.phar update
```

If you need to install a Plugin which isn't available on WordPress Packagist or that you have to build a custom plugin, add it into the project root in wp-content, it is symlinked into the WordPress install, so you can develop or make it available to WordPress.

### Themes Install

Now let's install a Themes, we do the same, adding it to our composer.json and run composer update.

```
"wpackagist-theme/vanilla": "1.3.5"
```

Update the theme in th build.properties and build.local.properties

```
wp.theme=vanilla
```

And let's activate it:

```
./vendor/bin/phing wp-theme-activate 
```

### Developing a dumy plugin

Let's develop a dumy plugin to show how it works, for this we create a dumy-plugin plugin into the root wp-content/plugins.

```
mkdir ./wp-content/plugins/dumy-plugin
```

And add a dumy-plugin.php file with the following:

```
<?php
    /*
    Plugin Name: dumy-plugin 
    Plugin URI:
    Description: A Dumy plugin 
    Author: John Doe
    Version: 1.0
    Author URI: https://www.johndoe-dumy-plugin.com
    */
?>
```

Open your browser to http://192.168.99.100/wp-admin/plugins.php and your plugin is displayed.

Add it to your build.properties and build.local.properties:

```
wp.plugins=business-profile,contact-form-7,updraftplus,w3-total-cache,wordpress-https,dumy-plugin
```

And let's activate it:

```
./vendor/bin/phing wp-plugins-activate
```

Same apply to the theme development. Be sure to only commit your themes and plugins of the root wp-content folder, not the ones installed by composer. 

So we have our base WordPress development ready, now we are going to create our staging environment on AWS Elastic BeanStalk.

## AWS Elastic BeanStalk Staging environment 

### Set-up the AWS environment accounts

AWS recommends the separation of responsibilities, and therefore you should create a separate AWS account for every environment you might require.

This tutorial explains the deployment in a staging environment. 

So let's start and create an AWS account for your testing environment.

**Set up a new account:**

1. Open [https://aws.amazon.com/](https://aws.amazon.com/), and then choose *Create an AWS Account*.
2. Follow the online instructions.

### Set-up the package S3 bucket

Let's first create an S3 bucket for your WP backup.

**Create a bucket**

1. Sign-in to the AWS Management Console and open the Amazon S3 console at https://console.aws.amazon.com/s3.
2. Click *Create Bucket*.
3. In the *Create a Bucket* dialog box, fill in the *Bucket Name*. In this example we will use **"my-wordpress-site-backup"**.
4. In the *Region* box, select a region from the drop-down list.
5. Click Create.

When Amazon S3 has successfully created your bucket, the console displays your empty bucket **"my-wordpress-site-backup"** in the Bucket panel.

### Creating the EC2 Key Pair

To create your key pair using the Amazon EC2 console

1. Open the Amazon EC2 console at https://console.aws.amazon.com/ec2/.
2. In the navigation pane, under *NETWORK & SECURITY*, choose *Key Pairs*.
3. Enter a name for the new key pair in the Key pair name field of the *Create Key Pair* dialog box, and then choose *Create*.
The private key file is automatically downloaded by your browser. The base file name is the name you specified as the name of your key pair, and the file name extension is *.pem*. Save the private key file in a safe place.
4. If you will use an SSH client on a Mac or Linux computer to connect to your Linux instance, use the following command to set the permissions of your private key file so that only you can read it :

```bash
chmod 400 my-key-pair.pem
```

### Set-up your Elastic BeanStalk Application

**To create a new application**

1. Open the Elastic Beanstalk console.
2. Choose Create New Application.
3. Enter the name of the application: **my-wordpress-site** 
4. Then click Next.

### Create your Application Environment 

**To launch an environment** 

1. Open the Elastic Beanstalk console.
2. Choose our application: **my-wordpress-site**
3. In the upper right corner, choose Create New Environment from the Actions menu.
4. Choose **Web server** environment types.
5. For Platform: PHP
6. For App code, choose Sample application.
7. Choose Configure more options.
8. Configuration presets: Low cost (Free Tier eligible)
9. Select ** Environment settings** and fill in the following:
  1. Name: staging
  2. Domain: my-wordpress-site
10. Select **Software settings** and fill in the following:
  1. Environment properties:
  * ENVIRONMENT: staging
  * AUTH_KEY:
  * AUTH_KEY:
  * LOGGED_IN_KEY:
  * LOGGED_IN_SALT:
  * NONCE_KEY:
  * NONCE_SALT:
  * SECURE_AUTH_KEY:
  * SECURE_AUTH_SALT:
  * MYSQL_ADDON_DB: staging-my-wordpress-site-db
  * MYSQL_ADDON_HOST: staging-my-wordpress-site-db.ce7wdtyntw8p.<REGION>.rds.amazonaws.com
  * MYSQL_ADDON_USER: my-wordpress-site-user-db
  * MYSQL_ADDON_PASSWORD: <YOUR_DB_PASSWORD>
  * S3_BACKUP_URL:
  * S3_MEDIA_URL:
11. Select **Instances** and fill in the following:
  1. Root volume type: General Purpose (SSD)
  2. Size: 10 GB
12. Select **Security** 
  1. EC2 key pair: <YOUR_KEY_PAIR>
13. Select **Notifications**
  1. Email: <YOUR_EMAIL>
14. Select **Network**
  1. Select your default VPC in the Virtual private cloud (VPC)
  2. Check the Public IP address
  3. Select all the Instance subnets
  4. Instance security groups select:
  * rds-launch-wizard 
15. Do not configure the Database settings.

8. Choose Create environment.

### Set the Elastic Beanstalk environment variables

Elastic Beanstalk use exported environment variables, we will have to set these in Elastic Beanstalk Configuration Software Configuration Environment variables.
The S3_BACKUP_URL and S3_MEDIA_URL are buckets backup files generated by the wordpress UpdraftPlus - Backup/Restore, use them to restore your environment.

To avoid notice configure the environment variables on your development environment.

```
export MYSQL_ADDON_HOST=192.168.99.100
export MYSQL_ADDON_DB=wordpress
export MYSQL_ADDON_PASSWORD=password
export MYSQL_ADDON_USER=root
export S3_BACKUP_URL=s3://backup-bucket/db-backup-updraftplus.zip
export S3_MEDIA_URL=s3://backup-bucket/media-backup-updraftplus.zip
export ENVIRONMENT=develop
export AUTH_KEY=""
export SECURE_AUTH_KEY=""
export LOGGED_IN_KEY=""
export NONCE_KEY=""
export AUTH_SALT=""
export SECURE_AUTH_SALT=""
export LOGGED_IN_SALT=""
export NONCE_SALT=""
```

### Start docker-compose

```
docker-compose up
```

### Setup your wordpress

```
./vendor/bin/phing setup-dev
```

Open you browser to http://192.168.99.100

Run phing without any argugments to get the list of all the targets. 

### Running behat test

```
./vendor/bin/behat -c tests/behat.local.yml
```
