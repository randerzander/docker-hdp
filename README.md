# docker-hdp

Built and tested with the latest version of [Docker for Mac](https://docs.docker.com/engine/installation/mac/#/docker-for-mac) and CentOS. Older versions of Docker provided by docker-machine and/or Docker Toolbox will not work.

Trying this on Windows? Please [let me know](https://twitter.com/randerzander) how it works out.

## Project Goals:
1. Provide a reusable base with which to experiment with various Hadoop versions, its ecosystem, and its configs w/o VMs
2. Provide a pseudo-distributed Hadoop environment, because single node setups make bad assumptions about how software works in multi-node clusters.
3. Provide an excuse to learn & play with Docker

These containers are not pushed to DockerHub, thus you'll need to build them locally:
```
docker-compose -f examples/compose/single-container.yml build
```

A successful build looks like:
```
docker-hdp randy> docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hdp/node            latest              cacb20b1b0d3        15 seconds ago      7.682 GB
hdp/ambari-server   latest              b0fad41dd49c        15 minutes ago      2.492 GB
hdp/postgres        latest              ad42250d5c8b        23 minutes ago      320.2 MB
centos              6                   cf2c3ece5e41        3 weeks ago         194.6 MB
postgres            latest              7ee9d2061970        6 weeks ago         275.3 MB
```

## Running HDP 2.5:
To run 3 containers (postgres, ambari-server, and a "single container HDP cluster"):
```
docker-compose -f examples/compose/single-container.yml up
```

After a minute or so, you can access Ambari's Web UI at localhost:8080. Default User/PW is admin/admin.

### Container name and FQDN
- compose_ambari-server.dev_1 `ambari-server.dev`
- compose_postgres.dev_1 `postgres.dev`
- compose_dn0.dev_1 `dn0.dev`

## Using Ambari Blueprints:
To snapshot your cluster's configuration into a blueprint:
```
# You can extract a blueprint as soon as you click Deploy. No need to wait for install to complete.
curl --user admin:admin -H 'X-Requested-By:admin' localhost:8080/api/v1/clusters/dev?format=blueprint > examples/blueprints/single-container.json 
```

*Note*: I give Docker 7 cores and 14GB of RAM. If you're running with less, you should generate your own Ambari Blueprints with the recommendations Ambari provides (it should auto-detect your environment's available resources).

To submit your blueprint to Ambari and have it install your cluster:
```
# Can swap "single-container" for multi-container, or any type saved in examples/blueprints and examples/hostgroups
sh submit-blueprint.sh single-container examples/blueprints/single-container.json
```

There are additional blueprints for common test-beds in examples/blueprints, including Hive-LLAP and HBase-Phoenix.

## Notes:
1. Ambari, Hive, and Ranger dbs have been pre-created in the postgres database running at postgres.dev. To configure them in Ambari, set Postgres as the DB type and change the Database URL to point at postgres.dev (as depicted in screenshot below) and leave everything else as the default options. The password for the dbs are all "dev":
![hive-setup](/screenshots/hive-setup.png?raw=true)
2. The "node" container can be used for master, worker, or both types of services. The ambari-agent is configured to register with ambari-server.dev automatically, thus no SSH key setup is necessary. Use dn0.dev (and master0.dev if using multi-container):
![cluster-hosts](/screenshots/cluster-hosts.png?raw=true)
3. Yum packages for all HDP services have been pre-installed in the "node" container. This lets cluster install take place much faster at the expense of a spurious warning from Ambari during Host-Checks.
4. All Ambari and HDP repositories are downloaded at buildtime. The versions and URLs are specified in .env in the project's root
5. Docker for Linux is more restrictive about "su" use, which Ambari relies on heavily, thus examples/compose/single-container.yml and multi-container.yml images are marked "privileged:true". Read up on the implications.

## Helpful Hints:
If you HDFS having issues starting up/not leaving SafeMode, it's probably because docker-compose is re-using containers from a previous run.

To start with fresh containers, before each run do:
```
docker-compose -f examples/compose/multi-container.yml rm
Going to remove compose_ambari-server.dev_1, compose_dn0.dev_1, compose_master0.dev_1, compose_postgres.dev_1
Are you sure? [yN] y
Removing compose_ambari-server.dev_1 ... done
Removing compose_dn0.dev_1 ... done
Removing compose_master0.dev_1 ... done
Removing compose_postgres.dev_1 ... done
```

Docker for Mac sometimes has storage space problems. I recommend adding the following to your ~/.bash_profile and restarting terminal:
```
function docker-cleanup(){
 # remove untagged images  
 docker rmi $(docker images | grep none | awk '{ print $3}')
 # remove unused volumes  
 docker volume rm $(docker volume ls -q )  
 # `shotgun` remove unused networks
 docker network rm $(docker network ls | grep "_default")   
 # remove stopped + exited containers, I skip Exit 0 as I have old scripts using data containers.
 docker rm -v $(docker ps -a | grep "Exit [0-255]" | awk '{ print $1 }')
}
```

Run "docker-cleanup" if you run into Docker errors or "No space left on device" issues inside containers.

Since Hadoop UIs often link to hostnames, add the following to your hosts file:
```
echo "127.0.0.1 ambari-server ambari-server.dev" >> /etc/hosts
echo "127.0.0.1 master0 master0.dev" >> /etc/hosts
echo "127.0.0.1 dn0 dn0.dev" >> /etc/hosts
```

TODO:
1. Steps for using latest Docker 1.12 Swarm & Compose on multiple hosts
