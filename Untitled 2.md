mày viết cho t 1 cái script làm việc như sau:
- dùng python
- tìm kiếm tất cả các pod thuộc 1 nhóm namespace cố định chạy trên node có tên chứa *basev5*
- detect nó ra là workload kiểu deployment, hay stateful set, các loại khác bỏ qua
- kiểm tra tất cả các workload đó nếu đang không set nodeSelector và tolerations thì thực hiện patch nhẹ nodeSelector của nó thành app: non-authorized-pool
mục đích là để kill những thằng đang set nodepool không đúng tiêu chuẩn đã được cung cấp từ trước

```sh
sb-adapter-uat                Active   214d
sb-backendapi                 Active   532d
sb-backendapi-dev             Active   532d
sb-backendapi-r22             Active   532d
sb-backendapi-test            Active   532d
sb-bi-chatbot                 Active   117d
sb-bi-chatbot-dev             Active   290d
sb-botithd                    Active   532d
sb-botnvq                     Active   532d
sb-bpm-newlos-dev             Active   489d
sb-bpm-newlos-test            Active   489d
sb-bpm-zeebe-dev              Active   479d
sb-bpm-zeebe-uat              Active   423d
sb-bpmapi                     Active   532d
sb-bpmapi-dev                 Active   532d
sb-bpmapi-test                Active   532d
sb-cdp-dev                    Active   283d
sb-clearml                    Active   532d
sb-coreai-agent-dev           Active   26d
sb-cvat                       Active   532d
sb-dataflow                   Active   417d
sb-datahub-dt                 Active   18d
sb-ekyc-dev                   Active   532d
sb-ekyc-test                  Active   532d
sb-milvus                     Active   143d
sb-n8n-dev                    Active   297d
sb-nhs-fe                     Active   436d
sb-nhs-fe-dev                 Active   487d
sb-nhs-fe-test                Active   486d
sb-openlineage-dt             Active   18d
sb-openmetadata-dt            Active   18d
sb-paperless-ngx              Active   82d
sb-qdrant                     Active   80d
sb-seapay-dev                 Active   473d
sb-seapay-test                Active   473d
sb-trino                      Active   532d
sb-trino-dt                   Active   291d
sb-vvp                        Active   532d
sb-vvp-dtm                    Active   215d
sb-vvp-jobs                   Active   532d
```

```python
FIXED_NAMESPACES = [
    "sb-adapter-uat",
    "sb-backendapi",
    "sb-backendapi-dev",
    "sb-backendapi-r22",
    "sb-backendapi-test",
    "sb-bi-chatbot",
    "sb-bi-chatbot-dev",
    "sb-botithd",
    "sb-botnvq",
    "sb-bpm-newlos-dev",
    "sb-bpm-newlos-test",
    "sb-bpmapi",
    "sb-bpmapi-dev",
    "sb-bpmapi-test",
    "sb-cdp-dev",
    "sb-clearml",
    "sb-coreai-agent-dev",
    "sb-cvat",
    "sb-dataflow",
    "sb-datahub-dt",
    "sb-ekyc-dev",
    "sb-ekyc-test",
    "sb-milvus",
    "sb-n8n-dev",
    "sb-nhs-fe",
    "sb-nhs-fe-dev",
    "sb-nhs-fe-test",
    "sb-openlineage-dt",
    "sb-openmetadata-dt",
    "sb-paperless-ngx",
    "sb-qdrant",
    "sb-seapay-dev",
    "sb-seapay-test",
    "sb-trino",
    "sb-trino-dt",
    "sb-vvp",
    "sb-vvp-dtm",
    "sb-vvp-jobs",
]

```