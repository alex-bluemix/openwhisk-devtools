DOCKER_HOST_IP ?= $(shell echo ${DOCKER_HOST} | grep -o "[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}" || echo localhost)
PROJECT_HOME ?= openwhisk-master
WSK_CLI ?= $(PROJECT_HOME)/bin/wsk

OPEN_WHISK_DB_PROTOCOL ?= http
OPEN_WHISK_DB_HOST ?= $(DOCKER_HOST_IP)
OPEN_WHISK_DB_PORT ?= 5984
OPEN_WHISK_DB_PROVIDER ?= CouchDB
OPEN_WHISK_DB_USERNAME ?= whisk_admin
OPEN_WHISK_DB_PASSWORD ?= some_passw0rd
DB_SUBJECTS_DBS ?= subjects
OPEN_WHISK_DB_ACTIONS ?= whisk_actions
OPEN_WHISK_DB_GW ?= whisk_gateway

ifndef VERBOSE
.SILENT:
endif

# Quick-Start is a simple way to get started with OpenWhisk locally
#   1. at start it builds the project and the docker containers
#   2. then it starts all components using docker-compose
#   3. it runs a sample hello-world function
#   To stop and cleanup the environment use: make destroy
quick-start: download docker run quick-start-pause hello-world quick-start-info

.PHONY: download
download:
	rm -rf ./openwhisk-master*
	if [ "$(PROJECT_HOME)" = "openwhisk-master" ]; then \
	    curl -O ./openwhisk-master.tar.gz -L https://api.github.com/repos/openwhisk/openwhisk/tarball/master > ./openwhisk-master.tar.gz; \
	    mkdir openwhisk-master; \
	    tar -xvf ./openwhisk-master.tar.gz --strip 1 -C openwhisk-master; \
	else \
	     echo "Skipping downloading the code from git as PROJECT_HOME is not default:" $(PROJECT_HOME); \
	fi

.PHONY: quick-start-pause
quick-start-pause:
	echo "waiting for the Whisk invoker to come up ... "
	until $$(curl --output /dev/null --silent --head --fail http://$(DOCKER_HOST_IP):8085/ping); do printf '.'; sleep 5; done
	sleep 30

.PHONY: quick-start-info
quick-start-info:
	echo "$$(tput setaf 2)To invoke the function again use: $$(tput setaf 4)make hello-world$$(tput sgr0)"
	echo "$$(tput setaf 2)To stop openwhisk use: $$(tput setaf 4)make destroy$$(tput sgr0)"

docker:
	echo "building the docker images ... "
	cd $(PROJECT_HOME) && \
	    ./gradlew distdocker -x :core:swift3Action:distDocker -x :core:swiftAction:distDocker

.PHONY: run
run: check-required-ports setup start-docker-compose init-couchdb init-couchdb-actions init-whisk-cli

.PHONY: check-required-ports
check-required-ports:
	echo "checking required ports ... "
	for port in 80 443 2888 5984 8085 8888 9092 2888 8400 8500 8600 8302; do \
		pid=`lsof -Pi :$$port -sTCP:LISTEN -t` ; \
		if [ ! -z "$$pid" ];  then echo "$$(tput setaf 1)Port $$port is taken by PID:$$pid.$$(tput sgr0)"; exit 1; fi; \
	done
	echo " ... OK"

.PHONY: setup
setup:
	mkdir -p ~/tmp/openwhisk/apigateway/ssl
	cd $(PROJECT_HOME)/ansible/roles/nginx/files/ && ./genssl.sh $(DOCKER_HOST_IP)
	cp $(PROJECT_HOME)/ansible/roles/nginx/files/*.pem ~/tmp/openwhisk/apigateway/ssl
	cp -r ./apigateway/* ~/tmp/openwhisk/apigateway/

.PHONY: start-docker-compose
start-docker-compose:
	DOCKER_COMPOSE_HOST=$(DOCKER_HOST_IP) docker-compose --project-name openwhisk up 2>&1 > ~/tmp/openwhisk/docker-compose.log &

.PHONY: init-couchdb
init-couchdb:
	echo "waiting for the database to come up ... "
	until $$(curl --output /dev/null --silent --head --fail http://$(DOCKER_HOST_IP):5984/_all_dbs); do printf '.'; sleep 5; done

	echo "initializing the database ... "
	# the folder config/keys is referenced from createSubjects.sh
	mkdir -p db/config/keys
	cp $(PROJECT_HOME)/ansible/files/auth.* db/config/keys

	OPEN_WHISK_DB_PROVIDER=$(OPEN_WHISK_DB_PROVIDER) \
	    OPEN_WHISK_DB_PROTOCOL=$(OPEN_WHISK_DB_PROTOCOL) \
	    OPEN_WHISK_DB_HOST=$(OPEN_WHISK_DB_HOST) OPEN_WHISK_DB_PORT=$(OPEN_WHISK_DB_PORT) \
	    OPEN_WHISK_DB_USERNAME=$(OPEN_WHISK_DB_USERNAME) OPEN_WHISK_DB_PASSWORD=$(OPEN_WHISK_DB_PASSWORD) \
	    DB_SUBJECTS_DBS=$(DB_SUBJECTS_DBS) DB_WHISK_AUTHS=$(DB_SUBJECTS_DBS) \
	    PROJECT_HOME=$(PROJECT_HOME) \
	    db/createSubjects.sh

	# cleanup the files referenced by createSubjects.sh
	rm -rf db/config/keys

.PHONY: init-couchdb-actions
init-couchdb-actions:
	echo "initializing CouchDB Views ... "
	echo "" > $(PROJECT_HOME)/whisk.properties
	echo db.provider=$(OPEN_WHISK_DB_PROVIDER) >> $(PROJECT_HOME)/whisk.properties
	echo db.protocol=$(OPEN_WHISK_DB_PROTOCOL) >> $(PROJECT_HOME)/whisk.properties
	echo db.host=$(OPEN_WHISK_DB_HOST) >> $(PROJECT_HOME)/whisk.properties
	echo db.port=$(OPEN_WHISK_DB_PORT) >> $(PROJECT_HOME)/whisk.properties
	echo db.username=$(OPEN_WHISK_DB_USERNAME) >> $(PROJECT_HOME)/whisk.properties
	echo db.password=$(OPEN_WHISK_DB_PASSWORD) >> $(PROJECT_HOME)/whisk.properties
	echo db.whisk.actions=$(OPEN_WHISK_DB_ACTIONS) >> $(PROJECT_HOME)/whisk.properties
	echo db.whisk.apigw=$(OPEN_WHISK_DB_GW) >> $(PROJECT_HOME)/whisk.properties
	cd $(PROJECT_HOME)/ && tools/db/wipeTransientDBs.sh
	rm $(PROJECT_HOME)/whisk.properties

.PHONY: init-whisk-cli
init-whisk-cli:
	echo "waiting for the Whisk controller to come up ... "
	until $$(curl --output /dev/null --silent --head --fail http://$(DOCKER_HOST_IP):8888/ping); do printf '.'; sleep 5; done
	echo "initializing CLI ... "
	$(WSK_CLI) -v property set --namespace guest --auth `cat $(PROJECT_HOME)/ansible/files/auth.guest` --apihost $(DOCKER_HOST_IP):443 -i

.PHONY: stop
stop:
	DOCKER_COMPOSE_HOST=$(DOCKER_HOST_IP) docker-compose --project-name openwhisk stop

.PHONY: destroy
destroy: stop
	DOCKER_COMPOSE_HOST=$(DOCKER_HOST_IP) docker-compose --project-name openwhisk rm
	echo "cleaning other openwhisk containers started by the invoker ... "
	docker ps | grep whisk | awk '{print $$1}' | xargs docker stop | xargs docker rm
	echo "cleaning dangling docker volumes ... "
	docker volume ls -qf dangling=true | xargs docker volume rm
	rm -rf ~/tmp/openwhisk
	rm -rf ./openwhisk-master*

# This task runs a hello-world function
#   1. It creates the function
#   2. It executes it
#   3. At the end it deletes it
.PHONY: hello-world
hello-world: create-hello-world-function
	echo "invoking the hello-world function ... "

	echo "$$(tput setaf 4)adding the function to whisk ...$$(tput sgr0)"
	$(WSK_CLI) -i action create hello hello.js

	echo "$$(tput setaf 4)invoking the function ...$$(tput sgr0)"
	res=`$(WSK_CLI) -i action invoke hello --blocking --result` \
	    && echo "invokation result:" $$res \
	    && (echo $$res | grep "Hello, World") || ($(WSK_CLI) -i action delete hello && exit 1)

	echo "$$(tput setaf 1)deleting the function ...$$(tput sgr0)"
	$(WSK_CLI) -i action delete hello
	rm hello.js

.PHONY: create-hello-world-function
create-hello-world-function:
	echo "$$(tput setaf 2)creating the hello.js function ...$$(tput sgr0)"
	echo 'function main(params) {var name = params.name || "World"; return { payload:  "Hello, " + name + "!" }; }' > hello.js

# Using the hello-world function this task executes a performance test using Apache Benchmark
.PHONY: hello-world-perf-test
hello-world-perf-test: create-hello-world-function
	$(WSK_CLI) -i action create hello-perf hello.js

	docker run \
	    --net openwhisk_default \
	    --link controller jordi/ab ab -k -n 2000 -c 20 \
	    -m POST -H "Authorization:Basic MjNiYzQ2YjEtNzFmNi00ZWQ1LThjNTQtODE2YWE0ZjhjNTAyOjEyM3pPM3haQ0xyTU42djJCS0sxZFhZRnBYbFBrY2NPRnFtMTJDZEFzTWdSVTRWck5aOWx5R1ZDR3VNREdJd1A=" \
	            -H "Content-Type:application/json" \
	            http://controller:8888/api/v1/namespaces/guest/actions/hello-perf?blocking=true

	$(WSK_CLI) -i action delete hello-perf
	rm hello.js
