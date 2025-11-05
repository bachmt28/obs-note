```groovy
/**
 * Lấy 1 giá trị cụ thể từ Vault KV (v1 hoặc v2)
 * @param path - đường dẫn đến secret
 * @param credentialId - ID của Jenkins Vault Credential
 * @param key - tên key bên trong secret
 * @return giá trị tương ứng với key được chỉ định
 */

def getSecret(String path, String credentialId, String key) {
  def secretValue = null
  withVault(configuration: [
    vaultCredentialId: credentialId,
    vaultUrl: VAULT_URL
  ], vaultSecrets: [[
    path: path,
    secretValues: [[envVar: 'SECRET_VALUE', vaultKey: key]]
  ]]) {
    secretValue = env.SECRET_VALUE
  }
  return secretValue
}
  
/**
 * Lấy toàn bộ key-value trong một secret (KV)
 * @param path - đường dẫn đến secret\
 * @param credentialId - ID của Jenkins Vault Credential
 * @return map chứa toàn bộ dữ liệu
*/

def getSecrets(String path, String credentialId) {
  def secrets = [:]
  withVault(configuration: [
    vaultCredentialId: credentialId,
    vaultUrl: 'https://vault.example.com'
  ], vaultSecrets: [[
    path: path,
    secretValues: [[envVar: 'FULL_SECRET_JSON', vaultKey: '']]
  ]]) {
    secrets = readJSON text: env.FULL_SECRET_JSON
  }
  return secrets
}

  

/**

 * Gọi trực tiếp Vault HTTP API (dùng cho các engine không hỗ trợ plugin)

 * @param token - Vault Token

 * @param method - HTTP method: GET, POST, PUT...

 * @param path - Vault API path (không có /v1)

 * @param body - dữ liệu JSON nếu là POST/PUT

 * @return response body đã parse JSON

 */

def callVaultAPI(String token, String method, String path, Map body = [:]) {

  def response = httpRequest(

    url: "https://vault.example.com/v1/${path}",

    httpMode: method.toUpperCase(),

    customHeaders: [[name: 'X-Vault-Token', value: token]],

    contentType: 'APPLICATION_JSON',

    requestBody: (body ? writeJSON returnText: true, json: body : ''),

    validResponseCodes: '200:299'

  )

  return readJSON text: response.content

}

  

/**

 * Lấy dynamic secret (ví dụ: AWS IAM, database credential)

 * @param token - Vault Token

 * @param mountPath - path của engine dynamic (vd: aws, database)

 * @param roleName - tên role để cấp credential

 * @return credential được cấp phát

 */

def issueDynamicSecret(String token, String mountPath, String roleName) {

  return callVaultAPI(token, 'GET', "${mountPath}/creds/${roleName}")

}

  

/**

 * Lấy Vault token từ Jenkins Credential dạng "Secret Text"

 * @param credentialId - ID của Jenkins credential

 * @return token string

 */

def getTokenFromCredential(String credentialId) {

  def token = null

  withCredentials([string(credentialsId: credentialId, variable: 'VAULT_TOKEN')]) {

    token = env.VAULT_TOKEN

  }

  return token

}

  

/**

 * Ghi dữ liệu vào Vault KV engine (thường là v2)

 * @param token - Vault token có quyền ghi

 * @param path - path đầy đủ đến secret

 * @param dataMap - map dữ liệu cần ghi

 */

def writeKV(String token, String path, Map dataMap) {

  def payload = [data: dataMap]

  callVaultAPI(token, 'POST', path, payload)

}

  
  

/**

 * Ghi kubeconfig lấy từ Vault ra file tạm thời và trả về đường dẫn

 *

 * @param path         Đường dẫn KV secret trong Vault (VD: team-a/data/k8s-creds)

 * @param credentialId ID của Jenkins credential chứa Vault token (thường là kiểu Secret Text)

 * @param key          Tên key trong secret chứa nội dung kubeconfig (mặc định: "kubeconfig" hoặc tên cluster)

 * @param fileName     Tên file kubeconfig muốn ghi ra (mặc định: "kubeconfig")

 * @return             Đường dẫn tuyệt đối tới file kubeconfig đã ghi

 */

def writeKubeconfig(String path, String credentialId, String key = 'kubeconfig', String fileName = 'kubeconfig') {

  def kubeconfigContent = getSecret(path, credentialId, key)

  def filePath = "${env.WORKSPACE}/${fileName}"

  writeFile file: filePath, text: kubeconfigContent.trim()

  sh "chmod 600 ${filePath}"

  return filePath

}


def writeDockerConfig(String vaultPath, String credentialId, String vaultKey = 'config', String dir = '.docker') {

  def configContent = getSecret(vaultPath, credentialId, vaultKey)

  def configDir = "${env.WORKSPACE}/${dir}"

  sh """

    mkdir -p ${configDir}

    echo '${configContent.trim()}' > ${configDir}/config.json

    chmod 600 ${configDir}/config.json

  """

  return configDir

}
```
