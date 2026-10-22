.PHONY: jira-10 jira-8 posting.import posting.update

include .env

JIRA_SERVICE_10 := jira-software
JIRA_SERVICE_8 := jira-software-8

JIRA_10_SPEC := jira-10/jira_software_dc_10003_swagger.v3.json
JIRA_10_SPEC_URL := https://developer.atlassian.com/server/jira/platform/jira_software_dc_10003_swagger.v3.json
JIRA_10_OPENAPI := jira-10/jira-10.3.openapi.json

JIRA_8_WADL := jira-8/jira-rest-plugin.$(JIRA_8_VERSION).wadl
JIRA_8_WADL_URL := https://docs.atlassian.com/software/jira/docs/api/REST/$(JIRA_8_VERSION)/jira-rest-plugin.wadl
JIRA_8_OPENAPI := jira-8/jira-$(JIRA_8_VERSION).openapi.json

# Posting only applies per-operation security, imports null examples as literal nulls, drops
# non-scalar query params, and sends optional ones empty unless they're marked required: false.
OPENAPI_FIX := del(.servers) \
	| .components.securitySchemes = {basic: {type: "http", scheme: "basic"}} \
	| del(.. | objects | select(has("example") and .example == null) | .example) \
	| (.paths[][] | objects) |= (.security = [{basic: []}] \
		| (.parameters[]? | select(.in != "path")) |= (.required = false \
			| .schema |= if .type | IN("string", "integer", "number", "boolean") then del(select(.default == "") | .default) else {type: "string"} end))

jira-10 jira-8: jira-%:
	docker compose up -d $(JIRA_SERVICE_$*)

jira-%.posting:
	@if [ -z "$$(docker compose ps -q --status running $(JIRA_SERVICE_$*))" ]; then \
		echo "Jira $* is not running. Start it with: make jira-$*" >&2; \
		exit 1; \
	fi
	@[ -d jira-$*/endpoints ] || $(MAKE) -s posting.import
	@if [ -n "$$(docker ps -q --filter name=^posting-jira-$*$$)" ]; then \
		docker attach posting-jira-$*; \
	else \
		docker compose run --rm --name posting-jira-$* posting --collection jira-$*/endpoints --env jira-$*/jira-$*.env; \
	fi

jira-%.shell:
	@docker compose exec $(JIRA_SERVICE_$*) bash

posting.import:
	@for dir in jira-*/; do \
		rm -rf "$${dir}endpoints"; \
		docker compose run --rm -T posting import "$${dir}"*.openapi.json -o "$${dir}endpoints" || exit 1; \
		rm -f "$${dir}endpoints/"*.env; \
	done
	@# Posting puts a hash in the login variable names (OPENAPI_BASIC_<hash>_USERNAME).
	@find jira-*/endpoints -name '*.posting.yaml' -exec perl -pi -e 's/OPENAPI_BASIC_[0-9a-f]+_/OPENAPI_BASIC_/g' {} +

posting.update:
	curl -fsSL -o $(JIRA_10_SPEC) $(JIRA_10_SPEC_URL)
	jq -e '$(OPENAPI_FIX)' $(JIRA_10_SPEC) > $(JIRA_10_OPENAPI)
	curl -fsSL -o $(JIRA_8_WADL) $(JIRA_8_WADL_URL)
	docker run --rm -v "$(CURDIR)/jira-8:/jira-8:ro" -w /tmp node:24-alpine \
		npx -y api-spec-converter@2.12.0 --from=wadl --to=openapi_3 --syntax=json /$(JIRA_8_WADL) \
		| jq -e '$(OPENAPI_FIX)' > $(JIRA_8_OPENAPI)
	$(MAKE) -s posting.import
