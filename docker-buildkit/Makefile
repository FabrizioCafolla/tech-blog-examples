BUILDKIT_HOST_NAME := buildkitd
export BUILDKIT_HOST := docker-container://$(BUILDKIT_HOST_NAME)

CONTEXT := app
CACHE_FOLDER := .cache/$(CONTEXT)

.check:
	@if [ "$(CONTEXT)" == "" ]; then \
		echo "Invalid context"; \
		exit 1; \
	fi
	@if [[ "$(BUILDKIT_HOST)" != docker-container* ]]; then \
        echo "BuildKit deamon start outside"; \
	else \
		$(MAKE) start-buildkit-deamon ; \
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

.build-ctl: .check clean
	@rm -Rf $(CACHE_FOLDER)/$(CACHE_TYPE)
	@time buildctl build \
  		--frontend dockerfile.v0 \
		--local context=$(CONTEXT) \
    	--local dockerfile=./$(CONTEXT) \
		--opt build-arg:LABEL_TEST=test \
		--export-cache type=local,oci-mediatypes=true,image-manifest=true,dest=$(CACHE_FOLDER)/$(CACHE_TYPE),mode=$(CACHE_TYPE),compression=gzip \
		--import-cache type=local,src=$(CACHE_FOLDER)/$(CACHE_TYPE) \
		--output type=image,name=$(CONTEXT) \
		--opt attest:sbom= \
		--trace trace.tmp.json \
		--debug-json-cache-metrics cache-metrics.json
	@jq -s '.' trace.tmp.json > $(CACHE_FOLDER)/$(CACHE_TYPE)/trace.json ; rm trace.tmp.json
	@mv cache-metrics.json $(CACHE_FOLDER)/$(CACHE_TYPE)/cache-metrics.json

start-buildkit-deamon:
	@if [[ "$$(docker ps -f name=$(BUILDKIT_HOST_NAME) -q)" == "" ]]; then \
		echo "BuildKit daemon start docker container"; \
		docker run -d --name $(BUILDKIT_HOST_NAME) --privileged moby/buildkit:latest || docker rm -f $(BUILDKIT_HOST_NAME); \
	else \
		echo "BuildKit daemon container is already running."; \
	fi

clean:
	@echo "Cleaning cache"
	@buildctl prune --all
	@buildctl du

build-max:
	@$(MAKE) .build-ctl CACHE_TYPE=max

build-min:
	@$(MAKE) .build-ctl CACHE_TYPE=min

compare-cache: .check extract-cache-min extract-cache-max
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

extract-cache-min:
	@$(MAKE) .extract-cache CACHE_TYPE=min

extract-cache-max:
	@$(MAKE) .extract-cache CACHE_TYPE=max
