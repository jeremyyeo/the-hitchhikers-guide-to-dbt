---
---

# dbt Cloud Egress IP as json

Retrieving dbt Cloud (Platform) egress IPs as json.

https://docs.getdbt.com/docs/cloud/about-cloud/access-regions-ip-addresses

```sh
# Install "jq" to process json.

$ curl -s https://cloud.getdbt.com/api/v2/constants/ | jq '.["data"]["environment_configuration"]["egress_ips"]'
[
  "52.45.144.63",
  "54.81.134.249",
  "52.22.161.231",
  "52.3.77.232",
  "3.214.191.130",
  "34.233.79.135"
]

$ curl -s https://emea.dbt.com/api/v2/constants/ | jq '.["data"]["environment_configuration"]["egress_ips"]'
[
  "3.123.45.39",
  "3.126.140.248",
  "3.72.153.148"
]

$ curl -s https://c1.jp1.dbt.com/api/v2/constants/ | jq '.["data"]["environment_configuration"]["egress_ips"]'
[
  "13.115.236.233",
  "54.238.211.79",
  "35.76.76.152"
]
```
