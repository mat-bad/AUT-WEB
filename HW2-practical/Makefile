SHELL := /bin/bash
.ONESHELL:
.DEFAULT_GOAL := up
.PHONY: up down token test

SUB ?= alice
TTL ?= 3600

# Ports/addresses are never hard-coded: they are read from .env. Recursive (=) on purpose,
# so they are looked up lazily, after .env has been created.
envval        = $(shell sed -n 's/^$(1)=//p' .env 2>/dev/null)
HTTP_PORT     = $(call envval,HTTP_PORT)
UPSTREAM_PORT = $(call envval,UPSTREAM_PORT)
LAN_ADDRESS   = $(or $(call envval,LAN_ADDRESS),127.0.0.1)
BASE_URL     ?= http://localhost:$(HTTP_PORT)

# base64url encoding: stdin -> stdout
B64URL = openssl base64 -A | tr '+/' '-_' | tr -d '='

# $(call jwt,<sub>,<ttl-seconds>) prints an HS256 JWT signed with the key in .env.
# A negative TTL gives an already-expired token.
define jwt
( hexkey=$$(sed -n 's/^JWT_SIGN_KEY=//p' .env | tr -d '\n' | openssl base64 -d -A | od -An -vtx1 | tr -d ' \n'); \
  now=$$(date +%s); \
  h=$$(printf '%s' '{"alg":"HS256","typ":"JWT"}' | $(B64URL)); \
  p=$$(printf '{"sub":"%s","exp":%s}' '$(1)' "$$((now + $(2)))" | $(B64URL)); \
  s=$$(printf '%s.%s' "$$h" "$$p" | openssl dgst -sha256 -mac HMAC -macopt hexkey:$$hexkey -binary | $(B64URL)); \
  printf '%s.%s.%s' "$$h" "$$p" "$$s" )
endef

# create .env once, with a random signing key
.env:
	@key=$$(openssl rand -base64 32)
	sed "s|^JWT_SIGN_KEY=.*|JWT_SIGN_KEY=$$key|" .env.example > .env

up: .env
	docker compose up -d --build

down:
	docker compose down

token: .env
	@$(call jwt,$(SUB),$(TTL))
	echo

test: .env
	@pass=0; fail=0
	check() { if [ "$$2" = "$$3" ]; then echo "PASS  $$1"; pass=$$((pass+1)); else echo "FAIL  $$1 (expected $$2, got $$3)"; fail=$$((fail+1)); fi; }
	code()  { curl -s -o /dev/null -w '%{http_code}' "$$@"; }
	good=$$( $(call jwt,alice,3600) )
	expired=$$( $(call jwt,alice,-60) )
	forged="$${good%.*}.AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"

	check "no token -> 401"                      401 "$$(code $(BASE_URL)/get)"
	check "garbage token -> 401"                 401 "$$(code -H 'Authorization: Bearer nope' $(BASE_URL)/get)"
	check "forged signature -> 401"              401 "$$(code -H "Authorization: Bearer $$forged" $(BASE_URL)/get)"
	check "expired token -> 401"                 401 "$$(code -H "Authorization: Bearer $$expired" $(BASE_URL)/get)"
	check "token in query string ignored -> 401" 401 "$$(code "$(BASE_URL)/get?access_token=$$good")"
	check "valid token -> 200"                   200 "$$(code -H "Authorization: Bearer $$good" $(BASE_URL)/get)"
	check "valid token, POST -> 200"             200 "$$(code -X POST -d x=1 -H "Authorization: Bearer $$good" $(BASE_URL)/post)"
	check "legacy service reachable on LAN address without a token" 200 "$$(code --max-time 3 http://$(LAN_ADDRESS):$(UPSTREAM_PORT)/get)"

	echo; echo "$$pass passed, $$fail failed"
	[ $$fail -eq 0 ]
