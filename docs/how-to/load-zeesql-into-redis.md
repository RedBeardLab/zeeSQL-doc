# How to load zeeSQL in Redis

After you get the [zeeSQL module](get-zeesql.md) you need to load it up in Redis.

If you are using zeeSQL from the docker image, this step is not necessary, since the image is already set up to work with zeeSQL out of the box.

If you are running zeeSQL from scratch or you are setting up your own infrastructure, then these steps are necessary.

All these steps assume that you have saved the zeeSQL module in `/home/user/zeeSQL.so`.

## Load from the command line

The simplest way to load the module in Redis is to pass the module as a flag when starting up the Redis instance.

You just need to provide the `--loadmodule` flag with the correct path.

For instance:

```
$ redis-server --loadmodule /home/user/zeeSQL.so
```

## Load from config

A more structured way to load the module is to use the default configuration file of Redis.

In the Redis configuration file, the `loadmodule` directive is available.

It is sufficient to use that directive passing the path of the module:

```
################################## MODULES #####################################

# Load modules at startup. If the server is not able to load modules
# it will abort. It is possible to use multiple loadmodule directives.
#

loadmodule /home/user/zeeSQL.so 

```

## Load with Redis running

The last option is to issue the `MODULE LOAD` command at runtime against Redis.

```
127.0.0.1:6379> MODULE LOAD /home/user/zeeSQL.so
8:M 06 Mar 2021 14:33:04.923 * Module 'rediSQL' loaded from /home/user/zeeSQL.so
OK
```

This will load the module exactly like the two other systems.

## About zeeSQL

zeeSQL is a Redis Module that provides SQL capabilities to Redis. It allows the creation and management of several SQL databases, each one independent from the other. Moreover, zeeSQL provides out-of-the-box [secondary indexes](../secondary-indexes.md) capabilities, allowing fast and easy search by value in Redis.
