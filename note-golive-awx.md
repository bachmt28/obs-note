## Cập nhật liên quan đến job stop start nhóm service way4

```yaml
JT mẫu:  http://172.28.199.21:32553/#/templates/job_template/118/details
playbook:
  git: https://gitlab-internal.seabank.com.vn/system/ops/sb-ansible-awx.git
  branch: main
  path: 
  - playbooks\w4\epin_app_start.yaml
  - playbooks\w4\epin_app_stop.yaml
  - playbooks\w4\ns_way4-app_start.yaml
  - playbooks\w4\ns_way4-app_stop.yaml
  - playbooks\w4\ws_way4-app_start.yaml
  - playbooks\w4\ws_way4-app_stop.yaml
  - playbooks\w4\ns_way4-app_start.yaml
  - playbooks\w4\ns_way4-app_stop.yaml
```

## Job nâng cấp nếu chuyển -> awx
### manual step
1. chuẩn bị bộ cài từ filesrv
2. copy vào folder 

vậy giờ t muốn như này 


```yaml
sequence:
	Preflight
	parallel:
		Checkov
		Fortify
		Custom Build
		sequence:
			Docker Build
			parallel:
				sequene:
					Helm Deploy
					Zap Scan
			Prisma Image Scan
```
