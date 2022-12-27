Zitified Beats
==============

This repo is a fork of [elastic/beats](https://github.com/elastic/beats). Its purpose to produce Zitified versions of
beats executables (filebeat, metricbeat, etc). Produced binaries can be used as drop-in replacements into existing beats
installations.

Note: Additional configuration is required to connect to collectors over OpenZiti overlay network.

# Building Executables

- Clone this repo to your/build machine. `zitify` [branch](https://github.com/openziti-test-kitchen/beats/tree/zitify)
- Standard Go build process:
  ```console
  # example build filebeat binary
  $ mkdir bin
  $ go build -o bin ./filebeat
  ```

# Ziti Configuration

1. Obtain an OpenZiti network in one of the following ways:
  - [quickstart](https://openziti.github.io/docs/quickstarts/network/)
  - Ziti Edge Developer Sandbox - [ZEDS](https://zeds.openziti.org)
  - Free to use (limited resources) [Teams Network](https://nfconsole.io/signup), hosted by NetFoundry

2. Create Logstash Ziti service.
   - follow sample [quickstart](https://openziti.github.io/docs/quickstarts/services/ztha). Note: skip client tunneler
     setup. This approach will use app-embedded zero trust SDK.
   - decide how logstash service is hosted:
      - hosting tunneler
      - [zitified logstash beats pluging](https://github.com/openziti-test-kitchen/logstash-input-zitibeats/tree/zitify)
   - on the beats client side the service should have an intercept configuration like this:
     ```json
     // intercept.v1
     {
       "protocols":["tcp"],
       "addresses":["beats.logstash.ziti"],
       "portRanges":[{"low":5044, "high":5044}]
      }
     ```

3. create beats endpoint with appropriate access to logstash service
   - create and enroll endpoint https://openziti.github.io/docs/core-concepts/identities/overview
   - configure policies https://openziti.github.io/docs/core-concepts/security/authorization/policies/overview


4. Configure beats to use identity and logstash service
   - assume your beats identity is stored in `beats.json` file
   - modify beats configuration (filebeat.yml, metricbeat.yml, etc) with intercepted logstash address
     ```yaml
       output.logstash:
           # The Logstash hosts
           hosts: ["beats.logstash.ziti:5044"]
     ```
5. Full configuration script:
   ```
   ziti edge create config beats.logstash.intercept intercept.v1 '{
    "protocols":["tcp"],
    "addresses":["beats.logstash.ziti"],
    "portRanges":[{"low":5044,"high":5044}]
    }'

   # logstash/zitibeats service
   ziti edge create service beats.logstash --configs beats.logstash.intercept

   # logstash identity
   ziti edge create identity user logstash -o logstash.jwt
   ziti edge enroll -j logstash.jwt -o logstash.json

   ziti edge create service-policy beats.logstash.bind Bind --service-roles="@beats.logstash" --identity-roles="@logstash"

   ziti edge create service-policy beats.logstash.dial Dial --service-roles="@beats.logstash" --identity-roles="#beat-agent"

   # create beats client identity
   ziti edge create identity user beatz -o beatz.jwt -a beat-agent
   ziti edge enroll -j beatz.jwt -o beatz.json

   ```
7. Run zitified beats agent using the drop-in executable
```console
$ ZITI_IDENTITIES=beats.json filebeat -c filebeat.yml
```
