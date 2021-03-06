SHELL=cmd

# include make_env

.PHONY: build push pull restart run stop login clean release tag

MAKEFILE_JUSTNAME = $(firstword $(MAKEFILE_LIST))
MAKEFILE_COMPLETE = $(CURDIR)/$(MAKEFILE_JUSTNAME)

$(info DIRECTORY $(CURDIR))

# Image and binary can be overidden with env vars.
# Hub user / repo name
DOCKER_IMAGE ?= workinghandguard/jenkins
# DOCKER_TAG ?= default
CONTAINER_NAME ?= jenkins

# Get the latest commit.
GIT_COMMIT = $(strip $(shell git rev-parse --short HEAD))

# Get the version number from the code
###
# CODE_VERSION = $(strip $(shell cat VERSION))
### 
#WIN
###
CODE_VERSION = $(strip $(shell type VERSION))
###
#

# Find out if the working directory is clean
GIT_NOT_CLEAN_CHECK = $(shell git status --porcelain)

ifneq (x$(GIT_NOT_CLEAN_CHECK), x)
	DOCKER_TAG_SUFFIX = -dirty
endif

##
## Make is very touchy about spaces and tabs, it treats indented lines as commands, so you need to remove them. That's why $(error) commands aren't formatted
##
	# ifneq (x$(GIT_NOT_CLEAN_CHECK), x)
	# 	$(error You are trying to release a build based on a dirty repo) - wrong
	#$(error You are trying to release a build based on a dirty repo) - right
	# endif

###	@:
###	@make
###	etc.
# It means "don't echo this command on the output." So this rule is saying "execute the shell command : and don't echo the output.
# Of course the shell command : is a no-op, so this is saying "do nothing, and don't tell." 
# The make(1) syntax is the use of an action starting with @, which is simply not to echo the command. 
# So a rule like
###	always:
###	@echo this always happens
# won't emit
###	echo this always happens
###	this always happens

# If we're releasing to Docker Hub, and we're going to mark it with the latest tag, it should exactly match a version release
ifeq ($(MAKECMDGOALS),release)
	# Use the version number as the release tag.
	DOCKER_TAG = $(CODE_VERSION)

	ifndef CODE_VERSION
$(error You need to create a VERSION file to build a release)
	endif

	# When the value results from complex expansions of variables and functions, expansions you would consider empty may actually contain whitespace characters and thus are not seen as empty. 
	# However, you can use the strip function to avoid interpreting whitespace as a non-empty value. 
	ifeq ($(CODE_VERSION),)
$(error VERSION file not found)
	endif

	# See what commit is tagged to match the version
	# VERSION_COMMIT = $(strip $(shell git rev-list $(CODE_VERSION) -n 1 | cut -c1-7))
	VERSION_COMMIT = $(strip $(shell git rev-list $(CODE_VERSION) -n 1 --abbrev-commit)) #finally...
	# WRONG! VERSION_COMMIT = $(strip $(shell for /f "tokens=7" %f in ('more') do @echo %f))

	ifneq ($(VERSION_COMMIT), $(GIT_COMMIT))
$(error You are trying to push a build based on commit $(GIT_COMMIT) but the tagged release version is $(VERSION_COMMIT))
	endif

	# Don't push to Docker Hub if this isn't a clean repo
	ifneq (x$(GIT_NOT_CLEAN_CHECK), x)
$(error You are trying to release a build based on a dirty repo)
	endif
else

	ifndef CODE_VERSION
$(error You need to create a VERSION file to build a release)
	endif

	# When the value results from complex expansions of variables and functions, expansions you would consider empty may actually contain whitespace characters and thus are not seen as empty. 
	# However, you can use the strip function to avoid interpreting whitespace as a non-empty value. 
	ifeq ($(CODE_VERSION),)
$(error VERSION file not found)
	endif

	# Add the commit ref for development builds. Mark as dirty if the working directory isn't clean
	DOCKER_TAG = $(CODE_VERSION)-$(GIT_COMMIT)$(DOCKER_TAG_SUFFIX)

endif

default: build

# Build Docker image
build: docker_build output

# Build and push Docker image
release: docker_build docker_push output

push:
	$(info Make: Pushing "$(DOCKER_TAG)" tagged image.)
	docker push $(DOCKER_IMAGE):$(DOCKER_TAG)

pull:
	$(info Make: Pulling "$(DOCKER_TAG)" tagged image.)
	docker pull $(DOCKER_IMAGE):$(DOCKER_TAG)

restart:
	$(info Make: Restarting "$(DOCKER_TAG)" tagged container.)
	@make -s -f "$(MAKEFILE_JUSTNAME)" stop
	@make -s -f "$(MAKEFILE_JUSTNAME)" run

clean:
	docker system prune --volumes --force

login:
	$(info Make: Login to Docker Hub.)
	docker login -u $(DOCKER_USER) -p $(DOCKER_PASS)

# 
# echo mySecret | docker secret create my_secret -
# 
# docker secret create my_secret ./secret.json
# 
# docker secret create --label env=dev \ 
# --label rev=20170324 \ 
# my_secret ./secret.json
# 

tag:
	$(info Make: Tagging image with "$(DOCKER_TAG)".)
	docker tag $(DOCKER_IMAGE):latest $(DOCKER_IMAGE):$(DOCKER_TAG)

# 	-it \
#	-v /var/run/docker.sock:/var/run/docker.sock \
# -p 8080:8080 \

run:
	$(info Make: Starting "$(DOCKER_TAG)" tagged container.)
	docker run -d \
	-p 50000:50000 \
	--restart unless-stopped \
	--name $(CONTAINER_NAME) \
	--mount source=jenkins-log,target=/var/log/jenkins \
	--mount source=jenkins-data,target=/var/jenkins_home \
	--network jenkins-net \
	$(DOCKER_IMAGE):$(DOCKER_TAG)

# --mount source=jenkins-log,target=/var/log/jenkins \
# --mount source=jenkins-data,target=/var/jenkins_home \
# OR
# -v ${PWD}/jenkins_home:/var/jenkins_home \
# -v ${CURDIR}/jenkins_home:/var/jenkins_home \

# In DinD way you can reuse outside docker cache sharing -v /var/lib/docker:var/lib/docker.
# In DooD way some many scenarios include to share -v /var/lib/jenkins:/var/lib/jenkins.

# -d \	
# &&\
# docker logs -f $(CONTAINER_NAME)

stop:
	$(info Make: Stopping "$(DOCKER_TAG)" tagged container.)
	docker stop $(CONTAINER_NAME)
	docker rm $(CONTAINER_NAME)

# Build Docker image
# mount "C:\Users\Username\Documents\Project" /home/Username/Project
# --build-arg BUILD_DATE=`date -u +%d.%m.%Y-%H:%M:%S,%2N` \
# -f Dockerfile .
### win
# --build-arg BUILD_DATE=`echo %date%-%time%` \
# --build-arg BUILD_DATE=`powershell get-date -format "{dd.MM.yyyy-HH:mm::ss.ff}"` \

### lin
# docker_build: Dockerfile
# 	$(info Make: Building "$(DOCKER_TAG)" tagged images.)
# 	docker build \
# 	--build-arg VERSION=$(CODE_VERSION) \
# 	--build-arg BUILD_DATE=`date -u +%d.%m.%Y-%H:%M:%S,%2N` \
# 	--build-arg VCS_URL=`git config --get remote.origin.url` \
# 	--build-arg VCS_REF=$(GIT_COMMIT) \
# 	-t $(DOCKER_IMAGE):$(DOCKER_TAG) \
# 	-f Dockerfile .
# 	@make -s clean

### win

docker_build: Dockerfile
	$(info Make: Building "$(DOCKER_TAG)" tagged images.)
	docker build \
	--build-arg VERSION="$(CODE_VERSION)" \
	--build-arg BUILD_DATE="`echo %date%-%time%`" \
	--build-arg VCS_URL="`git config --get remote.origin.url`" \
	--build-arg VCS_REF="$(GIT_COMMIT)" \
	-t "$(DOCKER_IMAGE):$(DOCKER_TAG)" \
	-f Dockerfile .
	@$(MAKE) --silent -f "$(MAKEFILE_JUSTNAME)" clean

# Automake cannot handle the concept of makefiles with alternate filenames, mainly because in order to generate the correct code, it would need to know about them.
# make will look for the make‐files GNUmakefile, makefile, and Makefile

docker_push:
# Tag image as latest
	docker tag $(DOCKER_IMAGE):$(DOCKER_TAG) $(DOCKER_IMAGE):latest
# Push to DockerHub
	@make -s -f "$(MAKEFILE_JUSTNAME)" push
	docker push $(DOCKER_IMAGE):latest

output:
	@echo Docker Image: $(DOCKER_IMAGE):$(DOCKER_TAG)