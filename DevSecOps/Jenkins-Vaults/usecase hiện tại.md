
| no  | use case                     | khả năng thực hiện | ghi chú                                                                                                                                                                                                              |
| --- | ---------------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | credential git kéo library   | no                 | chỉ có thể cấu hình trực tiếp                                                                                                                                                                                        |
| 2   | git kéo code                 | yes                | đang binding thông qua code và cred                                                                                                                                                                                  |
| 3   | secret tương tác với fortify | yes                | nhưng sẽ phải dùng hoàn toàn script và command thông qua vault plugin, fortify plugin trực tiếp không có<br>Tóm lại là bỏ không dùng plugin của jenkins mà phải cli hoá                                              |
| 4   | credential tương tác với k8s | yes                | dùng vault plugin như bình thường                                                                                                                                                                                    |
| 5   | secret token để gửi webex    | yes                | dùng vault plugin như bình thường                                                                                                                                                                                    |
| 6   | docker push                  | yes                | dùng vault plugin như bình thường                                                                                                                                                                                    |
| 7   | các case khác                |                    | plugin vault của jenkins sẽ giống như vault cli nhưng được giản lược hơn để dễ đưa vào declarative pipeline, chứ không tích hợp sẵn như các plugin của hệ thống, nên auto-revoke gần như khó thực hiện 1 cách native |

## PIPELINE Mẫu 

```groovy
@Library('jenkins-library') _
// Những thứ cần chú ý thay đổi khi clone pipeline 
// 1. secretToken của trigger webhook
// 2. env gitlab repo 
// 3. Các parameter nếu cần tương ứng với môi trường deploy
// 4. sửa lại đường dẫn/tên của script pipeline trong giao diện cấu hình pipeline trên jenkins

pipeline {
agent {label 'local-agent'}
options {
  // disableConcurrentBuilds() //Only 1 build run at the sametime. current build wait previous build finish to run
  buildDiscarder logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '3', numToKeepStr: '20')
}
triggers {
  gitlab branchFilterType: 'All', excludeBranchesSpec: '', includeBranchesSpec: '', noteRegex: 'Jenkins please try a build from gitlab webhook', pendingBuildName: '', secretToken: 'u2S2L22XR4bWsTyLGVgSDwuA4DRRMFVq', skipWorkInProgressMergeRequest: true, sourceBranchRegex: '', targetBranchRegex: '', triggerOnMergeRequest: false, triggerOpenMergeRequestOnPush: 'never'
}
environment {
  HTTPS_PROXY='http://dc2-proxyuat.seauat.com.vn:8080'
  NO_PROXY="gitlab-internal.seabank.com.vn,localhost,gke-dc2-nexus-devtestuat.seauat.com.vn,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,*.seabank.com.vn, *.seauat.com.vn, connectgateway.googleapis.com,199.36.153.8/30,gke-dtu-rancher.seauat.com.vn"
  GITLAB_REPOSITORY_URL='https://gitlab-internal.seabank.com.vn/bi_nlp2dashboard_prj/nlp2db-upgrade'
  VAULT_URL='https://vault-demo.seauat.com.vn'
}
parameters { 
  string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'branch chỉ định sẽ sử dụng nếu env gitlabBranch không có') 
  string(name: 'APP_LABEL', defaultValue: 'nlp2dashboard-service', description: 'Nhãn gắn vào trên workload, phục vụ cho quản trị, routing...')
  string(name: 'REGISTRY', defaultValue: 'dev-bi-docker.seauat.com.vn', description: 'Local repo image')
  string(name: 'NAMESPACE', defaultValue: 'sb-bi-chatbot-dev', description: 'namespace deploy workload')
  string(name: 'EVRM', defaultValue: 'dev', description: 'Biến dùng để chỉ định foler compile và ghép vào tên workload')
  string(name: 'VERSION', defaultValue: 'main', description: 'namespace deploy workload')
  string(name: 'DEPLOY_KIND', defaultValue: 'Deployment', , description: 'workload type StatefulSet or Deployment, case sensitive')
  string(name: 'GATEWAY_DNS', defaultValue: 'dev-data-assist.seauat.com.vn', , description: 'dns routing of workload, apply on service mesh')
  string(name: 'CONTEXT_PREFIX', defaultValue: '/nlp2db', , description: 'Path routing of workload, apply on service mesh')
  string(name: 'fortify_AppName', defaultValue: 'BI-API')
  string(name: 'fortify_AppName', defaultValue: '1.0')
  string(name: 'FORTIFY_TIMEOUT', defaultValue: '30')
  text(name: 'k8sClusterID', defaultValue: 'gke-dc1dtu-usrcl')
}
stages {
//// === ////
    stage ('Clean Workspace & Check Env'){ steps { script{
        cleanWs()
        lib.printGitlabEnv()
    }}}
    stage ('Pull Code'){ steps { script{
        lib.checkoutAndMerge("${BRANCH}", "${GITLAB_REPOSITORY_URL}", vaultPath: 'bi/data/gitlab')
    }}}
    stage ('Install & Package'){ steps { script {
        s4.dockerBuildDev(vaultPath: 'bi/data/docker', vaultKey: "${params.REGISTRY}")
    }}}
    stage ('Compile & Deploy '){ steps { script {
        s5.compileWorkloadK8S("${params.EVRM}")
        s5.deployWorkloadK8S("${params.EVRM}", vaultPath: 'bi/data/k8s', '' )
    }}}
    
  }
}
```