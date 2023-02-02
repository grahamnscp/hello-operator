# Replicated Vendor CLI
#
app_slug := "${REPLICATED_APP}"
RELEASE_CHANNEL=Unstable
PROMOTE_CHANNEL=Beta

createapp:
	@replicated app create ${app_slug}

list-releases:
	@replicated release ls --app ${app_slug}

lint:
	@replicated release lint --app $(app_slug) --yaml-dir manifests

release:
	@replicated release create \
		--auto -y \
		--yaml-dir manifests \
		--promote $(RELEASE_CHANNEL)

release-noauto:
	@replicated release create \
		--yaml-dir manifests \
		--promote $(RELEASE_CHANNEL)

release-beta:
	@replicated release create \
		--auto -y \
		--yaml-dir manifests \
		--promote Beta

promote-release:
	$(eval SEQUENCE=$(shell sh -c "replicated release ls --app ${REPLICATED_APP} --token ${REPLICATED_API_TOKEN} | grep ${RELEASE_CHANNEL}" | awk '{print $$1}'))
	$(eval LATEST_VERSION=$(shell sh -c "replicated --app ${REPLICATED_APP} --token ${REPLICATED_API_TOKEN} channel ls | grep ${PROMOTE_CHANNEL}" | awk '{print $$4}'))
	$(eval VERSION_MAJOR=$(shell sh -c "echo ${LATEST_VERSION}" | awk -F. '{printf "%s.%s",$$1,$$2}'))
	$(eval VERSION_MINOR=$(shell sh -c "echo ${LATEST_VERSION}" | awk -F. '{print $$3}'))
	$(eval VERSION_MINOR_INC=$(shell echo $$(($(VERSION_MINOR)+1))))
	$(eval NEW_VERSION=$(shell echo ${VERSION_MAJOR}.${VERSION_MINOR_INC}))
	@replicated release promote $(SEQUENCE) $(PROMOTE_CHANNEL) \
		--version $(NEW_VERSION) \
		--app $(REPLICATED_APP) \
		--token $(REPLICATED_API_TOKEN)
	@echo "Promoted release $(SEQUENCE) to channel $(PROMOTE_CHANNEL) as version $(NEW_VERSION)"

release-promote: release promote-release

