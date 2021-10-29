# Configuration

You may have wondered how running the `g` command without any configuration and
options still enabled Guillotina to connect and configure the database. This is because Guillotina will run without a user-defined configuration. By default, Guillotina will run with a DUMMY_FILE database which will save the database file locally.

In this section, we'll talk about working with the Guillotina configuration
system and configure Guillotina to run with a Postgresql database.  We will also show how to add additional applications to Guillotina.


## Getting started

Guillotina provides a command to bootstrap a configuration file for you.

```
g create --template=configuration
```

This will produce a `config.yaml` file in your current path. Inspect the file
to see what some of the default configuration options are.

## Modifying the configuration

A detailed list of configuration options and explanations can be found
in the [configuration section](../../installation/configuration.html) of the docs.


```eval_rst
.. note:: Guillotina also supports JSON configuration files
```

## Connecting Guillotina to PostgreSQL

In order to perform the examples in this training session, you will need a database.  Currently Guillotina only supports [PostgreSQL](https://www.postgresql.org/) for the training.  There are two options for linking Guillotina to a Postgres database engine:
  
* Use [Docker](https://www.docker.com/) to run Postgres (*e.g.*, on a *development* machine)
* Install and run Postgress on the same machine as Guillotina (*e.g.*, on a *virtual* machine)

The examples below work with Ubuntu 20.04.

### Docker

Docker is an easy option, especially if you have Docker installed on your system already.  However, the data in Docker is _volitile_.  **Once you power down the docker image, you will lose all of your data.**  If you want to make your data persistent for an extended period of time, you should install an instance of PostgreSQL on your machine (as described below).

####  Install Docker

For the Docker option, we presume that you will install Docker on the host machine.  If you don't have a working instance of Docker, there are several good [descriptions](https://docs.docker.com/engine/install/) for installing Docker on a variety of operating systems.  If you're using Ubuntu, a detailed (and well written) description of the Docker installation process is [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04).  However, for the impatient, here is an example using Ubuntu (on a device where Docker hasn't yet been installed):

Start with an updated system...

	$ sudo apt update

Install the necessary dependencies...

	$ sudo apt install apt-transport-https ca-certificates curl software-properties-common

Now, add the GPG key for the official Docker repository...

	$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

If all goes well, you should receive an `Ok` response.

Next, add the Docker repository to the APT sources ...

	$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"

Note, for other flavors of Ubuntu, you need to substitute "focal" for your particular flavor of Ubuntu and change the architecture ("arch=") as necessary.

At this point, you should check to ensure that you're going to get Docker from the official Docker repository, rather than the standard Ubuntu repository.  To do that, enter the following:

	$ apt-cache policy docker-ce

If all is correct, you should see an output something like:

	docker-ce:
		Installed: (none)
		Candidate: 5:19.03.9~3-0~ubuntu-focal
		Version table:
			5:19.03.9~3-0~ubuntu-focal 500
				500 https://download.docker.com/linux/ubuntu focal/stable amd64 Packages

You're almost there.  At this point, Docker is still not installed, but at least you know that you're about to install the correct one.  To install Docker, enter:

	$ sudo apt install docker-ce


#### Download and Run Postgres for Guillotina

Since we're using Docker, we won't be able to create any users.  Fortunately, Postgres comes pre-equipped with a user named **postgres**.  So we'll specify a password for **postgres** and designate a (new) database called _guillotina_.  Once Docker is installed, instantiate the PostgreSQL database with the following command:

	$ sudo docker run \
		-e POSTGRES_DB=guillotina \
		-e POSTGRES_USER=postgres \
		-e POSTGRES_PASSWORD=postgrespw \
		-p 127.0.0.1:5432:5432 \
		postgres:9.6

At this point, we need to tell Guillotina about the Postgres database and connection credentials.  To do that, we edit the `config.yaml` file to reflect the same information that we passed to PostgreSQL via Docker.  So for this example, the databases section within the `config.yaml` file should look like:

	databases:
		db:
			storage: postgresql
			dsn: postgresql://postgres:postgrespw@localhost:5432/guillotina
			readonly: false 

Once you've edited the `config.yaml` file, start Guillotina.

	$ g -c config.yaml
	
When you're finished, you can stop the Docker process with the `stop` command.  First, however, you need the container id.  To get that id, you list the active containers with the `ls` command:

	$ sudo docker container ls

You should get a response something like:

	CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                      NAMES
	03314d0a3ed3   postgres:9.6   "docker-entrypoint.s…"   10 minutes ago   Up 10 minutes   127.0.0.1:5432->5432/tcp   exciting_panini

In this case, the PostgreSQL container id is "03314d0a3ed3" (but yours will certainly be different).  Knowing that, we can stop PosgreSQL by issuing the following command:

	$ sudo docker stop 03314d0a3ed3	

### Install Postgres on the Machine

In the situation where Guillotina will be tested extensively over many days, it may be advantageous to install a non-volitile instance of PostgreSQL on the same machine as Guillotina.  

#### Installation

Install PostgreSQL from the [website](https://www.postgresql.org/download/) or use your distribution's package manager.  On Ubuntu, type in the following set of commands.  

First, let's make sure your system is up to date...

	$ sudo apt update
	
And now we install Postgres...

	$ sudo apt install postgresql postgresql-contrib

Since Guillotina should be configured to use Postgre under its own authority, we need to set up a _role_ (account) within Postgres to enable the authentication and authorization of Guillotina.  Upon installation, Postgres has one pre-defined role, namely **postgres**, which has to be used in order to create other roles.  So we now switch to the **postgres** account and generate the new user interactively.

	$ sudo -u postgres createuser --interactive

The first question that the database will ask is the name of the new user...

	Enter name of role to add:  

... and we will enter the name of the Guillotina user, in this case "guillotinauser".

	Enter name of role to add:  guillotinauser

The next question that needs to be addressed is whether you give Guillotina superuser rights.  For this training excercise, we will provider superuser rights, but your organization may not want that.  For training purposes, however, giving superuser status to Guillotina obviates some potential problems.

	Shall the new role be a superuser? (y/n)
	
... to which we answer ...

	Shall the new role be a superuser? (y/n) y

Now that the role has been created, we need to set a password for the Guillotina user.  To do that, however, we need to use a tool called `psql`.

	$ sudo -u postgres psql
	
Here, we're going to do two things, change the guillotinauser password, and create a database for Guillotina within Postgres.  For the database:

	postgres=#: CREATE DATABASE gdata;

If that works, you should get a `CREATE DATABASE` response.

For the user password, enter:

	postgres=# ALTER ROLE guillotinauser WITH PASSWORD 'guillotinapw';

If that works, you should get a `ALTER ROLE` response.  At this point, you are finished with the Postgres setup.  To exit the Postgres command line interface, use the `\q` command:

	postgres=# \q

At this point, we need to tell Guillotina about the Postgres database and connection credentials.  To do that, we edit the `config.yaml` file to reflect the same information that we passed to PostgreSQL through the command line.  So for this example, the databases section within the `config.yaml` file should look like:

	databases:
		db:
			storage: postgresql
			dsn: postgresql://guillotinauser:guillotinapw@localhost:5432/gdata
			readonly: false 

## Multiple configuration files

Sometimes it is helpful to have multiple configuration files.  Fortunately, the name of a configuration file is not limited to `config.yaml`.  To specify a configuration file other than the name `config.yaml`, you can use the `-c` or `--config` command line option.  For example:


```
g -c config-foobar.yaml
```

## Installing applications

Guillotina applications are python packages or modules that you install and then configure in your application settings (in the `config.yaml` file).

As an example, we'll activate swagger support in Guillotina.  Even though swagger has been packaged with Guillotina since version 5, we will add it explicitly in this example.

Edit your `config.yaml` file and make sure that `guillotina.contrib.swagger` is listed in your appliactions section.

```yaml
applications:
- guillotina.contrib.swagger
```

Finally, start Guillotina again and visit `http://localhost:8080/@docs`.


**References**

  - [Configuration Options](../../installation/configuration)


