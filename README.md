https://github.com/bhalgatsaket/MEDIAWIKI_ON_AMAZON_EC2.git

Deployment of MediaWiki on AWS cloud on Amazon Linux

- 1 haproxy load balancer.
- 2 Apache2 web servers running Mediawiki.
- 1 MySQL database server.

Total four Ubuntu 10.04 systems running in Amazon EC2.

Referance for cookbooks taken from git://github.com/opscode/php-quick-start.git (Not working because of many older cookbooks and configuration changes)

We're going to reuse a number of cookbooks from the [Cookbooks Community Site](http://cookbooks.opscode.com) to build the environment.Our Source code lives in **git**. 

Environment Setup
----

First, let's configure the local workstation.

### Shell Environment

Obtain the repository used for this guide. It contains all the components required. Use git:

    git clone git://github.com/bhalgatsaket/MEDIAWIKI_ON_AMAZON_EC2.git

### Chef and Knife

*Ubuntu/Debian users*: Install XML2 and XLST development headers on your system:

    sudo apt-get install libxml2-dev libxslt-dev

*All Users*: You'll need some additional gems for Knife to launch instances in Amazon EC2:

    sudo gem install knife-ec2
	
*CHEF* : Setup a Chef Client with knife.

    mkdir ~/MEDIAWIKI_ON_AMAZON_EC2/.chef
    cp ~/chef-repo/.chef/knife.rb ~/MEDIAWIKI_ON_AMAZON_EC2/.chef
    cp ~/chef-repo/.chef/USERNAME.pem ~/MEDIAWIKI_ON_AMAZON_EC2/.chef
    cp ~/chef-repo/.chef/ORGNAME-validator.pem ~/MEDIAWIKI_ON_AMAZON_EC2/.chef
	
Add the Amazon AWS credentials to the Knife configuration file.

    vi ~/MEDIAWIKI_ON_AMAZON_EC2/.chef/knife.rb
	
Add the following two lines to the end:

    knife[:aws_access_key_id] = "replace with the Amazon Access Key ID"
    knife[:aws_secret_access_key] =  "replace with the Amazon Secret Access Key ID"

Once the MEDIAWIKI_ON_AMAZON_EC2 and knife configuration is in place, we'll work from this directory.

    cd MEDIAWIKI_ON_AMAZON_EC2

### Amazon AWS EC2

In addition to the credentials, two additional things need to be configured in the AWS account.

Configure the default [security group](http://docs.amazonwebservices.com/AWSEC2/latest/DeveloperGuide/index.html?using-network-security.html) to allow incoming connections for the following ports.

* 22 - ssh
* 80 - haproxy load balancer
* 22002 - haproxy administrative interface
* 8080 - apache2 web servers running mod_php

Create an [SSH Key Pair](http://docs.amazonwebservices.com/AWSEC2/latest/DeveloperGuide/index.html?using-credentials.html#using-credentials-keypair) and save the private key in **~/.ssh**.

Acquire Cookbooks
----
Upload all the cookbooks to the Hosted Chef server.

    knife cookbook upload -a
	
Server Roles
------------

All the required roles have been created in the php-quick-start repository. They are in the **roles/** directory.

    base.rb
    mediawiki_database_master.rb
    mediawiki.rb
    mediawiki_load_balancer.rb

Upload all the roles to the Hosted Chef server.

    rake roles

Data Bag Item
----

The data bag name is **apps** and the item name is **mediawiki**. Upload this to the Hosted Chef server.

    knife data bag create apps
    knife data bag from file apps mediawiki.json
	
Launch Multi-instance Infrastructure
----
We will launch one database server, two application servers and one load balancer. 

First, launch the database instance.

    knife ec2 server create -r 'role[base],role[mediawiki_database_master]' -I ami-37af765e -G default -x ubuntu --node-name MySQL

Once the database master is up, launch one node that will create the database schema and set up the database with default data.

    knife ec2 server create -r 'role[base],role[mediawiki],recipe[mediawiki::db_bootstrap]' -I ami-37af765e -G default -x ubuntu --node-name Mediawiki01 

Launch the second application instance w/o the mediawiki::db_bootstrap recipe.

    knife ec2 server create -r 'role[base],role[mediawiki]' -I ami-37af765e -G default -x ubuntu --node-name Mediawiki02 

Once the second application instance is up, launch the load balancer.

    knife ec2 server create -r 'role[base],role[mediawiki_load_balancer]' -I ami-37af765e -G default -x ubuntu --node-name HAProxy

Once complete, we'll have four instances running in EC2 with MySQL, MediaWiki and haproxy up and available to serve traffic.

Verification
----

Knife will output the fully qualified domain name of the instance when the commands complete. Navigate to the public fully qualified domain name on port 80.

    http://ec2-xx-xxx-xx-xxx.compute-1.amazonaws.com/

The login is admin and the password is mediawiki.

You can access the haproxy admin interface at:

    http://ec2-xx-xxx-xx-xxx.compute-1.amazonaws.com:22002/
	
### Database Passwords

The data bag item for MediaWiki contains default passwords that should certainly be changed to something stronger.

The passwords in the MediaWiki Data Bag Item are set to the values show below:

    "mysql_root_password": {
      "_default": "mysql_root"
    },
    "mysql_debian_password": {
      "_default": "mysql_debian"
    },
    "mysql_repl_password": {
      "_default": "mysql_repl"
    },
    
To change the password to something stronger, modify **mysql_root**, **mysql_debian**, **mysql_repl** values. Something like the following secure passwords:

    vi data_bags/apps/mediawiki.json
    "mysql_root_password": {
      "_default": "super_s3cur3_r00t_pw"
    },
    "mysql_debian_password": {
      "_default": "super_s3cur3_d3b1@n_pw"
    },
    "mysql_repl_password": {
      "_default": "super_s3cur3_r3pl_pw"
    },

Once the entries are modified, simply load the data bag item from the json file:

    knife data bag from file apps mediawiki.json
