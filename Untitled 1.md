```groovy
    webexBotToken='NTVhYzk5ZDEtZjk4OC00ODU'
    def response = httpRequest  acceptType: 'APPLICATION_JSON_UTF8', contentType: 'APPLICATION_JSON_UTF8',
      customHeaders: [[name: 'Authorization', value: 'Bearer ' + webexBotToken]],
      httpProxy: (env.HTTPS_PROXY?.trim() ? env.HTTPS_PROXY : 'http://dc2-proxyuat:8080'),
      httpMode: 'POST', requestBody: patchOrgJSON.toString(),
      validResponseCodes: "1:399",
      timeout: 30, url: "https://webexapis.com/v1/messages"

    println("Response code: "+response.status)
```

Chỗ này t đang hardcode cái webexBotToken 

giờ muốn áp dụng sang vault
có 2 hướng là làm như trên, binding secret thông qua pipeline 

Tuy nhiên hàm webex gửi tin nhắn này đóng vai trò như 1 guard cuối cùng gửi token nội sinh cho các team có thẩm quyền
nếu binding kiểu đó thì không khác gì 'lậy ông tôi ở bụi này' cho đối tượng muốn khai thác 

Vì thế cần đổi sang kiểu withVault ngay trong code này 

```groovy
def withVaultEnv(Map cfg = [:], List secrets = [], Closure body) {
  def vaultUrl    = (cfg.vaultUrl ?: env.VAULT_URL)
  def credId      = (cfg.credId   ?: env.VAULT_CRED_ID)
  def engineVer   = (cfg.engineVersion ?: (env.VAULT_KV_ENGINE_VER ?: 2))

  if (!vaultUrl?.trim()) error "VAULT_URL is empty"
  if (!credId?.trim())   error "VAULT_CRED_ID is empty"
  if (!secrets || secrets.isEmpty()) error "vault secrets spec is empty"

  // Convert to withVault format + collect vars for masking
  def vaultSecrets = []
  def pairs = []

  secrets.each { s ->
    def path = (s.path ?: '').toString().trim()
    def key  = (s.key  ?: '').toString().trim()
    def varN = (s.var  ?: '').toString().trim()
    def ev   = (s.engineVersion ?: engineVer)

    if (!path) error "vault secret path is empty"
    if (!key)  error "vault secret key is empty"
    if (!varN) error "vault env var name is empty"

    vaultSecrets << [
      path: path,
      engineVersion: ev,
      secretValues: [[vaultKey: key, envVar: varN]]
    ]
  }

  withVault(
    configuration: [
      vaultUrl: "${vaultUrl}",
      engineVersion: engineVer,
      vaultCredentialId: "${credId}",
    ],
    vaultSecrets: vaultSecrets
  ) {
    // build mask pairs after env is populated
    secrets.each { s ->
      def varN = (s.var ?: '').toString().trim()
      // mask only if value exists to avoid null weirdness
      if (env."${varN}") {
        pairs << [var: varN, password: env."${varN}"]
      }
    }

    if (pairs) {
      wrap([$class: 'MaskPasswordsBuildWrapper', varPasswordPairs: pairs]) {
        body()
      }
    } else {
      body()
    }
  }
}

```

Hàm thì t có sẵn như này rồi . Giờ mày chỉ cần cải tiến giúp t đoạn 
thêm helper lấy webexBotToken từ vault thôi 