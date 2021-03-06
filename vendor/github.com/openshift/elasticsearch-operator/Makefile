CURPATH=$(PWD)
TARGET_DIR=$(CURPATH)/_output

GOBUILD=go build
BUILD_GOPATH=$(TARGET_DIR):$(TARGET_DIR)/vendor:$(CURPATH)/cmd

IMAGE_BUILDER_OPTS=
IMAGE_BUILDER?=imagebuilder
IMAGE_BUILD=$(IMAGE_BUILDER)
IMAGE_TAG?=docker tag

APP_NAME=elasticsearch-operator
APP_REPO=github.com/openshift/$(APP_NAME)
TARGET=$(TARGET_DIR)/bin/$(APP_NAME)
IMAGE_TAG=openshift/$(APP_NAME)
MAIN_PKG=cmd/$(APP_NAME)/main.go
RUN_LOG?=elasticsearch-operator.log
RUN_PID?=elasticsearch-operator.pid

# go source files, ignore vendor directory
SRC = $(shell find . -type f -name '*.go' -not -path "./vendor/*")

#.PHONY: all build clean install uninstall fmt simplify check run
.PHONY: all build clean fmt simplify run

all: build #check install

operator-sdk: get-dep
	@if ! type -p operator-sdk ; \
	then if [ ! -d $(GOPATH)/src/github.com/operator-framework/operator-sdk ] ; \
	  then git clone https://github.com/operator-framework/operator-sdk --branch master $(GOPATH)/src/github.com/operator-framework/operator-sdk ; \
	  fi ; \
	  cd $(GOPATH)/src/github.com/operator-framework/operator-sdk ; \
	  make dep ; \
	  make install || sudo make install || cd commands/operator-sdk && sudo go install ; \
	fi

imagebuilder:
	@if ! type -p imagebuilder ; \
	then go get -u github.com/openshift/imagebuilder/cmd/imagebuilder ; \
	fi

get-dep:
	@if ! type -p dep ; then \
		cd $(GOPATH) ; \
		if [ ! -d $(GOPATH)/bin ]; then \
			mkdir bin; \
			fi; \
			curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh ; \
	fi


build: $(SRC)
	@mkdir -p $(TARGET_DIR)/src/$(APP_REPO)
	@cp -ru $(CURPATH)/pkg $(TARGET_DIR)/src/$(APP_REPO)
	@cp -ru $(CURPATH)/vendor/* $(TARGET_DIR)/src
	@GOPATH=$(BUILD_GOPATH) $(GOBUILD) $(LDFLAGS) -o $(TARGET) $(MAIN_PKG)

clean:
	@rm -rf $(TARGET_DIR)

image: imagebuilder
	$(IMAGE_BUILDER) -t $(IMAGE_TAG) . $(IMAGE_BUILDER_OPTS)

test-e2e: image operator-sdk
	hack/test-e2e.sh

fmt:
	@gofmt -l -w $(SRC)

simplify:
	@gofmt -s -l -w $(SRC)

deploy: deploy-setup image deploy-image
	hack/deploy.sh
.PHONY: deploy

deploy-image:
	hack/deploy-image.sh
.PHONY: deploy-image

deploy-example:
	@oc create -n openshift-logging -f hack/cr.yaml
.PHONY: deploy-example

deploy-setup:
	EXCLUSIONS="05-deployment.yaml image-references" hack/deploy-setup.sh
.PHONY: deploy-setup

go-run: deploy deploy-example
	@ALERTS_FILE_PATH=files/prometheus_alerts.yml \
	RULES_FILE_PATH=files/prometheus_rules.yml \
	OPERATOR_NAME=elasticsearch-operator WATCH_NAMESPACE=openshift-logging \
	KUBERNETES_CONFIG=/etc/origin/master/admin.kubeconfig \
	go run cmd/elasticsearch-operator/main.go > $(RUN_LOG) 2>&1 & echo $$! > $(RUN_PID)

undeploy:
	hack/undeploy.sh
.PHONY: undeploy
