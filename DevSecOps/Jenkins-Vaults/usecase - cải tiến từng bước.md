# Hàm checkout and merge

```groovy
/**
 * Hàm checkout Git có sử dụng Git token từ Vault
 * Token được chèn vào URL và ẩn trong log nhờ wrap(MaskPasswordsBuildWrapper)
 */
def checkoutAndMerge(String branch, String repo, String vaultPath, String credentialId = 'vault-creds-cicd', String vaultKey = 'git_ci_token', String targetDir = null) {
  def gitToken = VaultHelper.getSecret(vaultPath, credentialId, vaultKey)
  def safeRepoUrl = "https://${gitToken}@${repo}.git"

  withEnv(["SAFE_GIT_URL=${safeRepoUrl}"]) {
    wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: [[var: 'SAFE_GIT_URL', password: gitToken]]]) {
      def checkoutStep = {
        checkout changelog: true, poll: true, scm: [
          $class: 'GitSCM',
          branches: [[name: "origin/${branch}"]],
          doGenerateSubmoduleConfigurations: false,
          extensions: [],
          submoduleCfg: [],
          userRemoteConfigs: [[
            url: env.SAFE_GIT_URL,
            name: 'origin',
            credentialsId: ''
          ]]
        ]
      }

      if (targetDir) {
        dir(targetDir) {
          checkoutStep()
        }
      } else {
        checkoutStep()
      }
    }
  }
}
```

---
# Cải tiến docker login
```groovy
def dockerBuildDev(String vaultPath, String credentialId = 'vault-creds-cicd', String vaultKey = 'config', String dir = '.docker') {
  try {
    echo "🚀 Building Docker image for ${env.BRANCH}"

    // Ghi config.json từ Vault ra thư mục .docker-config
    def configPath = VaultHelper.writeDockerConfig(
      vaultPath,    // path Vault
      credentialId,           // Vault Credential ID
      vaultKey,                       // key chứa config.json
      '.docker-config'                // thư mục tạm (tùy chọn)
    )

    def dockerfile = params.DOCKERFILE ?: 'Dockerfile'
    def imageTag = "${params.REGISTRY}/${env.APP_LABEL.toLowerCase()}:${params.VERSION}.${env.BUILD_NUMBER}.${env.BRANCH}"

    sh """
      docker --config ${configPath} build -f ${dockerfile} -t ${imageTag} .
      echo "📦 Image built: ${imageTag}"
      docker --config ${configPath} push ${imageTag}
      echo "✅ Docker push complete"
    """
  } catch (err) {
    if (currentBuild.result != "ABORTED") {
      currentBuild.result = "FAILURE"
    }
    throw err
  }
}

```
# Áp dụng vault để lấy kubeconfig

```groovy
def deployWorkloadK8S(ENVIRONMENT, String vaultPath, String credentialId = 'vault-creds-cicd') {
    try {
        def APP_NAME = params.VERSION?.trim() ? "${APP_LABEL}-${ENVIRONMENT}-${VERSION}" : "${APP_LABEL}-${ENVIRONMENT}"
        def clusterList = params.k8sClusterID.split("\n")
        
        echo "Triển khai lên ${clusterList.size()} cluster..."
        for (i = 0; i < clusterList.size(); i++) {
            def clusterKey = clusterList[i].trim()
            echo "▶️ Deploy to cluster key: ${clusterKey}"
            // Lấy kubeconfig từ Vault và ghi file tạm
            def kubeconfigPath = VaultHelper.writeKubeconfig(vaultPath, credentialId, clusterKey, ".kubeconfig-${clusterKey}")
            // Kustomize và apply workload
            
            sh "kubectl --insecure-skip-tls-verify kustomize ${BASE_PATH}/"
            sh "mkdir -p ${WORKSPACE}/redeploy/${NAMESPACE}/"
            sh "kubectl --insecure-skip-tls-verify kustomize ${ENV_PATH}/ > ${WORKSPACE}/redeploy/${NAMESPACE}/${APP_NAME}.yaml"
            echo "📄 Output manifest:"
            sh "cat ${WORKSPACE}/redeploy/${NAMESPACE}/${APP_NAME}.yaml"
            echo "🚀 Apply to ${clusterKey}..."
            sh "kubectl --insecure-skip-tls-verify --kubeconfig ${kubeconfigPath} apply -k ${ENV_PATH}/"
            
            // Cleanup
            sh "rm -f ${kubeconfigPath}"
        }
        
    } catch (err) {
        if (currentBuild.result != "ABORTED") {
            currentBuild.result = "FAILURE"
        }
        throw err
    }
}
```
---
# Cải tiến tương tác với fortify 

# Cải tiến hàm gửi webex
fd