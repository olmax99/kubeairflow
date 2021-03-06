define message1
	Environment variable stack_name is required. Not set.
        Use following command:
        "$$ export stack_name=<your stack>"

endef

ifndef stack_name
$(error $(message1))
endif
ifndef revision
revision := $(shell date --utc +%Y%m%dT%H%M%SZ)
endif


define getRef
$(shell aws cloudformation describe-stacks \
	--stack-name $(stack_name) \
	--query "Stacks[0].Outputs[?OutputKey=='$(1)'].OutputValue" \
	--output text)
endef

APPLICATION := $(call getRef,CodeDeployApplication)
DEPLOYMENT_GROUP := $(call getRef,CodeDeployDeploymentGroup)
DEPLOYMENTS_BUCKET := $(call getRef,DeploymentsBucket)

# ---------------- CodeDeploy-----------------------------------------

PACKAGE := $(stack_name)_$(revision).tgz


package:
	cd airflow && tar czf ../$(PACKAGE) .

upload: package
	aws s3 cp $(PACKAGE) s3://$(DEPLOYMENTS_BUCKET)

# Each deploy will trigger a new bundle as target revision
deploy: upload
	aws deploy create-deployment \
		--application-name $(APPLICATION) \
		--deployment-group-name $(DEPLOYMENT_GROUP) \
		--s3-location bucket=$(DEPLOYMENTS_BUCKET),bundleType=tgz,key=$(PACKAGE) \
		--deployment-config-name CodeDeployDefault.AllAtOnce \
		--file-exists-behavior OVERWRITE

# ---------------- Docker-Compose-------------------------

config:
	cp airflow/airflow.cfg airflow/airflow_prod.cfg
	mv airflow/airflow_dev.cfg airflow/airflow.cfg

compose: config
	docker-compose -f docker-compose.development.yml up --build

down:
	docker-compose -f docker-compose.development.yml down
	cp airflow/airflow.cfg airflow/airflow_dev.cfg
	mv airflow/airflow_prod.cfg airflow/airflow.cfg