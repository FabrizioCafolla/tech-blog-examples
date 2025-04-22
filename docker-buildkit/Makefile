BUILDKIT_HOST_NAME := buildkitd
export BUILDKIT_HOST := docker-container://$(BUILDKIT_HOST_NAME)

CONTEXT := app
IMAGE_FOLDER := .images/
DOCKERFILE := Dockerfile
CACHE_FOLDER := .cache/$(CONTEXT)
CACHE_CLEAN := true

.check:
	@if [ "$(CONTEXT)" == "" ]; then \
		echo "Invalid context"; \
		exit 1; \
	fi
	@if [[ "$(BUILDKIT_HOST)" != docker-container* ]]; then \
        echo "BuildKit deamon start outside"; \
	else \
		$(MAKE) buildkit-start-deamon ; \
	fi
	@echo "BuildKit daemon is running on $(BUILDKIT_HOST)"

.extract-cache: .check
	@if [ "$(CACHE_TYPE)" == "" ]; then \
		echo "Invalid cache type"; \
	fi
	@echo "Extracting cache: $(CACHE_TYPE)"
	@rm -Rf $(CACHE_FOLDER)/$(CACHE_TYPE)-extracted
	@for file in $$(ls $(CACHE_FOLDER)/$(CACHE_TYPE)/blobs/sha256); do \
		echo "Extracting $$file"; \
		mkdir -p $(CACHE_FOLDER)/$(CACHE_TYPE)-extracted/$$file; \
		tar -xzf $(CACHE_FOLDER)/$(CACHE_TYPE)/blobs/sha256/$$file -C $(CACHE_FOLDER)/$(CACHE_TYPE)-extracted/$$file 2> /dev/null || echo "" ; \
	done
	@echo "Extracted cache to $(CACHE_FOLDER)/$(CACHE_TYPE)-extracted\n\n"

.buildctl: .check clean
	@rm $(CACHE_FOLDER)/cache-metrics-$(CACHE_TYPE).json
	@rm $(CACHE_FOLDER)/trace-$(CACHE_TYPE).json
	@time buildctl build \
  		--frontend dockerfile.v0 \
		--local context=$(CONTEXT) \
    	--local dockerfile=./$(CONTEXT) \
		--opt filename=$(DOCKERFILE) \
		--opt build-arg:LABEL_TEST=test \
		--export-cache type=local,oci-mediatypes=true,image-manifest=true,dest=$(CACHE_FOLDER)/$(CACHE_TYPE),mode=$(CACHE_TYPE),compression=zstd,compression-level=3 \
		--import-cache type=local,src=$(CACHE_FOLDER)/$(CACHE_TYPE) \
		--output type=oci,name=$(CONTEXT):$(CACHE_TYPE),dest=$(IMAGE_FOLDER)$(CONTEXT)-$(CACHE_TYPE).tar \
		--trace $(CACHE_FOLDER)/trace-$(CACHE_TYPE).json \
		--debug-json-cache-metrics $(CACHE_FOLDER)/cache-metrics-$(CACHE_TYPE).json

.load-image:
	@if [ "$(CACHE_TYPE)" == "" ]; then \
		echo "Invalid cache type"; \
	fi
	@echo "Loading image $(CONTEXT)-$(CACHE_TYPE).tar"
	@docker load -i $(IMAGE_FOLDER)$(CONTEXT)-$(CACHE_TYPE).tar

create-test-file:
	@dd if=/dev/urandom of=$(CONTEXT)/test-file bs=1G count=1

buildkit-start-deamon:
	@if [[ "$$(docker ps -f name=$(BUILDKIT_HOST_NAME) -q)" == "" ]]; then \
		echo "BuildKit daemon start docker container"; \
		docker run -d --name $(BUILDKIT_HOST_NAME) --privileged moby/buildkit:latest || docker rm -f $(BUILDKIT_HOST_NAME); \
	else \
		echo "BuildKit daemon container is already running."; \
	fi

buildkit-down-deamon:
	@if [[ "$$(docker ps -f name=$(BUILDKIT_HOST_NAME) -q)" != "" ]]; then \
		echo "Stopping BuildKit daemon"; \
		docker stop $(BUILDKIT_HOST_NAME); \
		docker rm $(BUILDKIT_HOST_NAME); \
	else \
		echo "BuildKit daemon container is not running."; \
	fi

clean:
	@if [ "$(CACHE_CLEAN)" == "true" ]; then \
		echo "Cleaning cache"; \
		rm -Rf $(CACHE_FOLDER)/$(CACHE_TYPE); \
		buildctl prune --all; \
		buildctl du; \
		make create-test-file; \
	fi

build-max:
	@$(MAKE) .buildctl CACHE_TYPE=max

build-min:
	@$(MAKE) .buildctl CACHE_TYPE=min

load-image-min:
	@$(MAKE) .load-image CACHE_TYPE=min

load-image-max:
	@$(MAKE) .load-image CACHE_TYPE=max

cache-compare: .check cache-extract-min cache-extract-max
	@echo "Comparing max and min cache modes"
	@echo "Size of max cache:"
	@du -sh $(CACHE_FOLDER)/max-extracted
	@echo "Number of layers in max cache: $$(ls -l $(CACHE_FOLDER)/max-extracted | wc -l)"
	@echo "\nSize of min cache:"
	@du -sh $(CACHE_FOLDER)/min-extracted
	@echo "Number of layers in min cache: $$(ls -l $(CACHE_FOLDER)/min-extracted | wc -l)"
	@echo "\nLayers that include 'test-file'"
	@find .cache/app/max-extracted -type f -name "test-file"
	@find .cache/app/min-extracted -type f -name "test-file"

cache-extract-min:
	@$(MAKE) .extract-cache CACHE_TYPE=min

cache-extract-max:
	@$(MAKE) .extract-cache CACHE_TYPE=max

init: clean buildkit-start-deamon
	@mkdir -p $(CACHE_FOLDER)
	@mkdir -p $(IMAGE_FOLDER)
