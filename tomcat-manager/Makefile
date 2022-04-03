CONTEXT="."
VERSION=1.0
IMAGE_NAME=rajasoun/tomcat-manager:$(VERSION)

# HELP
# This will output the help for each task
# thanks to https://marmelab.com/blog/2016/02/29/auto-documented-makefile.html
.PHONY: help

help: ## This help.
	@awk 'BEGIN {FS = ":.*?## "} /^[a-zA-Z_-]+:.*?## / {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST)

.DEFAULT_GOAL := help

# DOCKER TASKS
# Build the container
build: ## Build the container
	docker build --build-arg VERSION=$(VERSION) -t $(IMAGE_NAME) $(CONTEXT)

login: ## Login To Docker
	cat token.txt | docker login --username rajasoun --password-stdin

push: login ## Push to Docker Hub
	docker push $(IMAGE_NAME)

run: ## Run the Container
	docker run --rm -p8080:8080 --name tomcat $(IMAGE_NAME) 