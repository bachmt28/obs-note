## Jenkinsfile 

```groovy
#!/usr/bin/env groovy

/*
 * ZAP Security Scanning Pipeline - SeABank Security Team
 * Target: SeANet via Apigee API Gateway (172.18.253.8)
 * Proxy: McAfee WebProxy (10.17.12.60) - Noted
 */

pipeline {
    agent any
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '30', artifactNumToKeepStr: '10'))
        timestamps()
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    parameters {
        string(name: 'TARGET_URL', defaultValue: 'https://uat-ebank.seauat.com.vn', description: 'URL mục tiêu SeANet')
        string(name: 'CRED_ID', defaultValue: 'zap-auth-pentest', description: 'ID Credentials trong Jenkins')
        string(name: 'CONFIG_FILE', defaultValue: 'seanet/config/automation-template-seanet.yaml', description: 'Đường dẫn file YAML')
    }
    
    environment {
        TIMESTAMP = new Date().format('yyyyMMdd_HHmm')
        SCAN_ID = "ZAP_${BUILD_NUMBER}_${TIMESTAMP}"
        REPORT_DIR = "${WORKSPACE}/reports/${SCAN_ID}"
        ZAP_IMAGE = "ghcr.io/zaproxy/zaproxy:latest"
        ZAP_CONTAINER = "zap-executor-${BUILD_NUMBER}"
        
        // Cập nhật IP theo đính chính của User
        APIGEE_GW_IP = "172.18.253.8" 
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    echo ">>> Khởi tạo môi trường Build #${BUILD_NUMBER}"
                    cleanWs()
                    checkout scm
                    sh "mkdir -p ${REPORT_DIR} ${WORKSPACE}/temp"
                }
            }
        }

        stage('Generate Configuration') {
            steps {
                script {
                    echo ">>> Đang chuẩn bị cấu hình cho SeANet..."
                    if (!fileExists(params.CONFIG_FILE)) {
                        error "Không thấy file: ${params.CONFIG_FILE}"
                    }
                    def template = readFile(params.CONFIG_FILE)
                    withCredentials([usernamePassword(credentialsId: params.CRED_ID, passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        def finalYaml = template
                            .replace('{{TARGET_URL}}', params.TARGET_URL)
                            .replace('{{USERNAME}}', USER)
                            .replace('{{PASSWORD}}', PASS)
                        writeFile file: "${WORKSPACE}/temp/automation.yaml", text: finalYaml
                    }
                }
            }
        }

        stage('Execute ZAP Scan') {
            steps {
                script {
                    echo ">>> Đang chạy quét tập trung vào Apigee Gateway (IP: ${APIGEE_GW_IP})..."
                    withCredentials([usernamePassword(credentialsId: params.CRED_ID, passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        try {
                            sh "chmod -R 777 ${REPORT_DIR}"
                            
                            sh """
                                docker run --name ${ZAP_CONTAINER} --rm \
                                    --add-host servicetest.seabank.com.vn:${APIGEE_GW_IP} \
                                    --add-host ebanknewtest.seabank.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebank.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host gc-api-uat.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebankbackend.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebankms1.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebankms2.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebankms3.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebankms4.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebankms5.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebankms6.seauat.com.vn:${APIGEE_GW_IP} \
                                    --add-host uat-ebankms7.seauat.com.vn:${APIGEE_GW_IP} \
                                    -v ${WORKSPACE}/temp/automation.yaml:/zap/wrk/automation.yaml:ro \
                                    -v ${REPORT_DIR}:/zap/wrk/reports:rw \
                                    -v ${WORKSPACE}/seanet/scripts:/zap/wrk/scripts:ro \
                                    -v ${WORKSPACE}/seanet/config:/zap/wrk/config:ro \
                                    ${ZAP_IMAGE} zap.sh -cmd \
                                    -autorun /zap/wrk/automation.yaml \
                                    -config script.globalvar.seanet.username=${USER} \
                                    -config script.globalvar.seanet.password=${PASS} \
                                    2>&1 | tee ${REPORT_DIR}/zap-console.log
                            """
                        } catch (Exception e) {
                            echo ">>> ZAP Scan hoàn tất. Kiểm tra kết quả trong báo cáo."
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // 1. Dọn dẹp container ZAP
                sh "docker rm -f ${ZAP_CONTAINER} 2>/dev/null || true"

                // 2. Xuất báo cáo HTML lên giao diện Jenkins
                if (fileExists("${REPORT_DIR}")) {
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: "${REPORT_DIR}",
                        reportFiles:'zap-report.html',
                        reportName: 'ZAP Security Report',
                        reportTitles: "SeANet Scan via Apigee GW - Build #${BUILD_NUMBER}"
                    ])
                }

                // 3. Lưu trữ artifacts
                archiveArtifacts artifacts: "reports/${SCAN_ID}/**/*", allowEmptyArchive: true

                // 4. GỬI EMAIL BÁO CÁO CHO TEAM DEV & ANTT
                def emailTo = "cuong.bt@seauat.com.vn" // Cập nhật email thực tế
                def emailSubject = "[DAST] Báo cáo quét bảo mật ZAP - ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}"
                def emailBody = """
                    <h2>Báo cáo kết quả quét bảo mật tự động DAST (OWASP ZAP)</h2>
                    <p>Dear Team ANTT,</p>
                    <p>Hệ thống OWASP ZAP đã hoàn tất việc rà quét bảo mật cho ứng dụng SeANet.</p>
                    <ul>
                        <li><b>Dự án:</b> ${env.JOB_NAME}</li>
                        <li><b>Lần quét (Build):</b> #${env.BUILD_NUMBER}</li>
                        <li><b>Trạng thái Pipeline:</b> ${currentBuild.currentResult}</li>
                        <li><b>URL truy cập hệ thống Jenkins:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></li>
                    </ul>
                    <p>Vui lòng xem file <b>zap-report.html</b> được đính kèm trực tiếp trong email này để rà soát chi tiết các cảnh báo (Alerts) và lỗ hổng bảo mật được phát hiện.</p>
                    <p>Trân trọng,</p>
                    <p>Email tự động từ Hệ thống</p>
                """

                // Lệnh gọi plugin gửi email
                emailext(
                    to: emailTo,
                    subject: emailSubject,
                    body: emailBody,
                    mimeType: 'text/html',
                    attachmentsPattern: "reports/${SCAN_ID}/zap-report.html"
                )        
            }
        }
    }
}
```

## automation-seaapp.yaml
```yaml
---
env:
  contexts:
    - name: "SeaApp-Context"
      urls:
        # Khai báo các tên miền gốc của hệ thống SeaApp & API
        - "https://uat-nhsfe-gw.seauat.com.vn"
        - "https://uat-seatellergwint.seauat.com.vn"
        - "https://gwint-uat.seauat.com.vn"
      includePaths:
        # Bao gồm toàn bộ các API đi qua các Gateway
        - "https://uat-nhsfe-gw.seauat.com.vn/.*"
        - "https://uat-seatellergwint.seauat.com.vn/.*"
        - "https://gwint-uat.seauat.com.vn/.*"
      excludePaths:
        # Loại trừ các file tĩnh hoặc thư mục không cần thiết để tăng tốc độ quét
        - ".*\\.js"
        - ".*\\.css"
        - ".*\\.png"
        - ".*\\.jpg"
        - ".*\\.svg"
        - ".*\\.woff2?"
  parameters:
    failOnError: true
    failOnWarning: false
    progressToStdout: true

jobs:
  # 1. ĐƯA KỊCH BẢN ĐÁNH CHẶN (INTERCEPTOR SCRIPT)
  - type: script
    parameters:
      action: "add"
      type: "httpsender"
      engine: "Graal.js"
      name: "BearerTokenExtractor_SeaApp.js"
      file: "/zap/wrk/scripts/BearerTokenExtractor_SeaApp.js"
      
  # 2. ĐỌC DANH SÁCH API TỪ POSTMAN COLLECTION
  - type: postman
    parameters:
      collectionFile: "/zap/wrk/config/seaapp_postman_clean.json"
      
  # 3. KÍCH HOẠT RÀ QUÉT (ACTIVE SCAN)
  - type: activeScan
    parameters:
      context: "SeaApp-Context"
      policy: "API-Minimal"
      maxRuleDurationInMins: 5
      maxScanDurationInMins: 60
      addQueryParam: true
      defaultPolicy: "API-Minimal"
      delayInMs: 0
      handleAntiCSRFTokens: false
      injectPluginIdInHeader: false
      scanHeadersAllRequests: false
      threadPerHost: 5
      
  # 4. XUẤT BÁO CÁO HTML
  - type: report
    parameters:
      template: "traditional-html"
      reportDir: "/zap/wrk/reports"
      reportFile: "zap-report-seaapp"
      reportTitle: "ZAP Security Report - SeaApp (SeaTeller)"
      reportDescription: "Báo cáo kiểm thử bảo mật động DAST tự động cho ứng dụng SeaApp qua luồng CI/CD."
      displayReport: false
```

## seaapp_postman_clean.json
```json
{
    "info": {
        "name": "SeaApp_BSV_FE_API_Clean",
        "description": "Bản đồ API sạch phục vụ quét DAST POC cho ứng dụng SeaApp (SeaMileage). Đã cập nhật đầy đủ 13 APIs.",
        "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
    },
    "item": [
        {
            "name": "0. Login Trigger",
            "request": {
                "method": "POST",
                "header": [
                    { "key": "Content-Type", "value": "application/json" }
                ],
                "url": {
                    "raw": "https://uat-seatellergwint.seauat.com.vn/services/msuaa/api/authenticate/ldapAuthen",
                    "protocol": "https",
                    "host": ["uat-seatellergwint", "seauat", "com", "vn"],
                    "path": ["services", "msuaa", "api", "authenticate", "ldapAuthen"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{ \"note\": \"JS Script sẽ đè body chuẩn vào đây\" }"
                }
            }
        },
        {
            "name": "Lấy phân quyền (getPermission)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"BSV_FE_API\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"getPermission\",\n        \"data\": {\n            \"appID\": \"BSV\",\n            \"userName\": \"SEAMILEAGE.UAT.03\"\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Lấy danh sách tài khoản (getAllAccount)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"getAllAccount\",\n        \"data\": {\n            \"customerId\": null,\n            \"emailInputter\": \"itsolpa.10@seanettest.vn\",\n            \"businessRegNumber\": \"\",\n            \"cusEmail\": null,\n            \"cusPhone\": \"\",\n            \"accEmail\": \"\",\n            \"accPhone\": \"\",\n            \"cartId\": \"\",\n            \"status\": null,\n            \"fromTime\": \"\",\n            \"toTime\": \"\",\n            \"pageIndex\": 1,\n            \"pageSize\": 10\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Xem chi tiết một tài khoản (getAccountById)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"getAccountById\",\n        \"data\": {\n            \"accountId\": \"5d6028af-7999-4d26-9ecd-e302797ab58c\"\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Lấy thông tin doanh nghiệp (getCustomerById)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"getCustomerById\",\n        \"data\": {\n            \"businessRegNumber\": \"14936889\",\n            \"stringToken\": \"token_se_duoc_inject_tu_js\"\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Kiểm tra số điện thoại trùng lặp (checkPhone)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"checkPhone\",\n        \"data\": {\n            \"phoneList\": [\n                {\n                    \"phone\": \"0969805678\",\n                    \"customerId\": \"19198899\"\n                }\n            ]\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Kiểm tra email trùng lặp bên VNA (checkEmail)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"checkEmail\",\n        \"data\": {\n            \"emailList\": [\n                {\n                    \"email\": \"test@gmail.com\",\n                    \"customerId\": \"\"\n                }\n            ]\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Nhập liệu đơn (sendAccount)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"sendAccount\",\n        \"data\": {\n            \"action\": \"REGISTER\",\n            \"customerInfo\": {\n                \"customerId\": \"19198899\",\n                \"businessRegNumber\": \"19198899\",\n                \"customerName\": \"KHDN19198899\",\n                \"address\": \"DIA CHI KHDN19198899\",\n                \"lastName\": \"AAA\",\n                \"firstName\": \"DOANH\",\n                \"email\": \"19198899@oscvn.com\",\n                \"phone\": \"0901919883\",\n                \"accountBSV\": \"\",\n                \"cardId\": \"1342513421423\"\n            },\n            \"accountInfoList\": [\n                {\n                    \"title\": \"MRS\",\n                    \"familyName\": \"123123\",\n                    \"firstName\": \"123123\",\n                    \"dateOfBirth\": \"10/11/2025\",\n                    \"email\": \"123123@fg\",\n                    \"phone\": \"0969809939\"\n                },\n                {\n                    \"title\": \"MR\",\n                    \"familyName\": \"123123\",\n                    \"firstName\": \"13123\",\n                    \"dateOfBirth\": \"10/11/2025\",\n                    \"email\": \"123123@ffasdfr\",\n                    \"phone\": \"09775353454\"\n                }\n            ],\n            \"inputterInfo\": {\n                \"title\": \"phong.lh\",\n                \"departmentId\": \"\",\n                \"departmentName\": \"\",\n                \"email\": \"hue.lh@gmail.com\"\n            }\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Lấy thông tin ID thẻ (getCardID)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"getCardID\",\n        \"data\": {\n            \"customerId\": \"14936889\",\n            \"stringToken\": \"token_se_duoc_inject_tu_js\"\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Cập nhật thông tin tài khoản BSV (updateAccount - UPDATE)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"updateAccount\",\n        \"data\": {\n            \"action\": \"UPDATE\",\n            \"customerInfo\": {\n                \"customerId\": \"19198899\",\n                \"businessRegNumber\": \"19198899\",\n                \"customerName\": \"KHDN19198899\",\n                \"address\": \"DIA CHI KHDN19198899\",\n                \"lastName\": \"AAA\",\n                \"firstName\": \"DOANH\",\n                \"email\": \"19198899@oscvn.com\",\n                \"phone\": \"0901919883\",\n                \"accountBSV\": \"\",\n                \"cardId\": \"1342513421423\"\n            },\n            \"accountInfo\": {\n                \"accountId\": \"fc62a5de-b7c6-4b25-86d4-6a0cd1c3652d\",\n                \"title\": \"MRS\",\n                \"familyName\": \"NN\",\n                \"firstName\": \"MM\",\n                \"dateOfBirth\": \"10/11/2025\",\n                \"email\": \"a003@khdn.com\",\n                \"phone\": \"0969833666\"\n            },\n            \"inputterInfo\": {\n                \"title\": \"phong.lh\",\n                \"departmentId\": \"\",\n                \"departmentName\": \"\",\n                \"email\": \"hue.lh@gmail.com\"\n            }\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Phản hồi kết quả đăng ký TK BSV (updateAccount - FEEDBACK)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"updateAccount\",\n        \"data\": {\n            \"action\": \"FEEDBACK\",\n            \"accountInfo\": {\n                \"accountId\": \"482328e4-013f-4458-ae21-d8e9b586305e\",\n                \"status\": \"FAIL\",\n                \"note\": \"testseests-feedback\"\n            },\n            \"inputterInfo\": {\n                \"title\": \"phong.lh\",\n                \"departmentId\": \"\",\n                \"departmentName\": \"\",\n                \"email\": \"hue.lh@gmail.com\"\n            }\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Xem toàn bộ config (getAllConfigTime)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"getAllConfigTime\",\n        \"data\": {\n            \"pageSize\": 10,\n            \"pageIndex\": 1,\n            \"toEmail\": \"\",\n            \"hour\": \"\"\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Tạo mới config (saveConfigTime)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"saveConfigTime\",\n        \"data\": {\n            \"hour\": 16,\n            \"toEmail\": \"itsolpa.12@seanettest.vn\"\n        }\n    }\n}"
                }
            },
            "response": []
        },
        {
            "name": "Chỉnh sửa config (updateConfigTime)",
            "request": {
                "method": "POST",
                "header": [
                    {
                        "key": "Content-Type",
                        "value": "application/json"
                    },
                    {
                        "key": "x-api-key",
                        "value": "nhs67dc6cb2b92e07978a890db9aferee",
                        "type": "text"
                    }
                ],
                "url": {
                    "raw": "https://uat-nhsfe-gw.seauat.com.vn/SEAMILEAGE_FE_API/api/rest/process",
                    "protocol": "https",
                    "host": ["uat-nhsfe-gw", "seauat", "com", "vn"],
                    "path": ["SEAMILEAGE_FE_API", "api", "rest", "process"]
                },
                "body": {
                    "mode": "raw",
                    "raw": "{\n    \"header\": {\n        \"reqType\": \"REQUEST\",\n        \"api\": \"V1-FE-API-SEAMILEAGE\",\n        \"apiKey\": \"nhs67dc6cb2b92e07978a890db9aferee\",\n        \"priority\": \"3\",\n        \"channel\": \"BRANCH\",\n        \"subChannel\": \"BRANCH\",\n        \"context\": \"WEB\",\n        \"userID\": \"SEAMILEAGE.UAT.03\",\n        \"synasyn\": \"true\"\n    },\n    \"body\": {\n        \"authenType\": \"updateConfigTime\",\n        \"data\": {\n            \"configId\": \"052903aa-a5a7-4523-8378-8f25a6d019d9\",\n            \"hour\": 17,\n            \"toEmail\": \"itsolpa.10@seanettest.vn\",\n            \"isOpen\": false\n        }\n    }\n}"
                }
            },
            "response": []
        }
    ]
}
```

## BearerTokenExtractor_SeaApp.js
```js
// BearerTokenExtractor_SeaApp.js
function sendingRequest(msg, initiator, helper) {
    var url = msg.getRequestHeader().getURI().toString();
    var ScriptVars = Java.type('org.zaproxy.zap.extension.script.ScriptVars');

    // 1. XỬ LÝ GÓI TIN ĐĂNG NHẬP
    if (url.indexOf("/api/authenticate/ldapAuthen") > -1) {
        var username = ScriptVars.getGlobalVar("seaapp.username") || "SEAMILEAGE.UAT.03"; 
        var password = ScriptVars.getGlobalVar("seaapp.password") || "225588cC"; 

        var payloadObj = {
            "header": {
                "reqType": "REQUEST",
                "api": "MsUaa",
                "priority": "3",
                "channel": "SEATELLER",
                "subChannel": "SEATELLER",
                "context": "PC",
                "location": "0.0.0.0"
            },
            "body": {
                "authenType": "ldapAuthen",
                "data": {
                    "appId": "SEATELLER",
                    "username": username,
                    "password": password
                }
            }
        };

        msg.setRequestBody(JSON.stringify(payloadObj));
        msg.getRequestHeader().setContentLength(msg.getRequestBody().length());
        msg.getRequestHeader().setHeader("Content-Type", "application/json");
        msg.getRequestHeader().setHeader("Origin", "https://uat-nhsfe-gw.seauat.com.vn");
    } 

    // 2. XỬ LÝ CÁC API NGHIỆP VỤ
    else if (url.indexOf("/SEAMILEAGE_FE_API/") > -1) {
        var token = ScriptVars.getGlobalVar("seaapp.jwt.token");
        if (token) {
            msg.getRequestHeader().setHeader("Authorization", "Bearer " + token);
            
            var reqBodyStr = msg.getRequestBody().toString();
            if (reqBodyStr.indexOf("token_se_duoc_inject_tu_js") > -1) {
                var newReqBodyStr = reqBodyStr.replace("token_se_duoc_inject_tu_js", token);
                msg.setRequestBody(newReqBodyStr);
                msg.getRequestHeader().setContentLength(msg.getRequestBody().length());
            }
        }
        // Các IBM gateway Headers bắt buộc
        msg.getRequestHeader().setHeader("X-IBM-Client-Id", "1e61743c03d3952acfad9f0495dabf35");
        msg.getRequestHeader().setHeader("X-IBM-Client-Secret", "c7a98afb828d85a451019f3b252f458c");
        msg.getRequestHeader().setHeader("x-client-id", "AAC6fbseOuQGqEg5HPG8sq7oXAHjaxgwWzFWKTGAX7Fp2p5A");
    }
}

function responseReceived(msg, initiator, helper) {
    var url = msg.getRequestHeader().getURI().toString();
    var ScriptVars = Java.type('org.zaproxy.zap.extension.script.ScriptVars');

    if (url.indexOf("/api/authenticate/ldapAuthen") > -1) {
        var responseBody = msg.getResponseBody().toString();
        try {
            var jsonResp = JSON.parse(responseBody);
            if (jsonResp && jsonResp.body && jsonResp.body.data && jsonResp.body.data.id_token) {
                ScriptVars.setGlobalVar("seaapp.jwt.token", jsonResp.body.data.id_token);
            }
        } catch (e) {}
    }
}
```