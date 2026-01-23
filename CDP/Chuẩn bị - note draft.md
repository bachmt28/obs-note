

Chuẩn bị parcel 
- Cần lisence (hỏi lại a Dũng)
- BNH gửi lại doc chi tiết presiqute:
	- Service bắt buộc phải active trên các node 
	- hardening về mặt OS (kèm link docs tham khảo cụ thể từ hãng )
	- link down load chính xác từ các requirements
	
- Kèm cả phụ lục tinh chỉnh OS, alert, cache,... dựa theo kn của BNH
- 
SeABank sẽ xem và chuẩn bị môi trường dựa theo tài liệu của BNH gửi 

Phía a Mạnh cần , hiện tại 
- IceBerg

Cloudera support: dung.ba@seabank.com.vn / Qtht#2022
Cloudera.com


Link tham khảo:

Database Postgre Prepare
https://docs.cloudera.com/cdp-private-cloud-base/7.3.1/cdp-private-cloud-base-installation/topics/cdpdc-required-databases.html


Role node
https://docs.cloudera.com/cdp-private-cloud-base/7.3.1/cdp-private-cloud-base-installation/topics/cdpdc-runtime-cluster-hosts-role-assignments.html


```groovy
parameters {
  text defaultValue: '''repositories:
  dev-bi-docker.seauat.com.vn:
    frontend-service:
      rules:
        "main.*-dev": 2
        "main.*-test": 3
      default_keep: 1
    all-remain:
      rules:
        "main.*-dev": 2
        "main.*-test": 3
      default_keep: 1''', name: 'config'
}
```


```yaml
repositories:
  dev-bi-docker.seauat.com.vn:
    specific-workload:
      rules:
        "main.*-dev": 2
        "main.*-test": 3
      default_keep: 1
    all-remain:
      rules:
        "main.*-dev": 2
        "main.*-test": 3
      default_keep: 1
```