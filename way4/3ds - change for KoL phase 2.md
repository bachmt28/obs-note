## Tổng quan
- Dựa trên tóm lược điều tra lỗi của a Vinh, các thay đổi nóng đã được thực hiện trên hệ thống 3ds VISA
```properties
##  Nguyên nhân lỗi
- giao dịch có OTP khi chưa thay đổi cấu hình: - chạy 1 tps --> đến 40 giao dịch là lỗi. Log không ghi nhận xử lý. 
- luồng không có OTP : chạy ổn định ở 5tps. phát sinh có một file client.txt tứ thời bắt đầu triển khai version 2.0, chương trình đọc vào để xử lý rule bị lỗi)
## Đề xuất
- luồng không có OTP : chạy ổn định ở 10tps/12tps  , đã test đến 15tps và phát sinh nhiều lỗi (10%)
- luồng có OTP : chạy ổn định ở 7tps, từ 10tps phát sinh lỗi (liên quan đến safenet : reset connection)
```
- Change này được tạo để áp dụng thực hiện tiếp trên 3ds MASTER
## Thực hiện thay đổi
---
### ACS web
- Sửa tham số java của ACS web trong file **`/opt/3dsacs/appserver/applications/way4acsweb/conf/config.properties`**
```
jvm_params=-Xmx16g -Xms16g
```
- Thêm tham số trong file **`/opt/3dsacs/appserver/applications/way4acs2/webapps/way4acs2/WEB-INF/conf/config.xml`**
nội dung như phần `<!-- Them moi cho KOL 121125 -->` như sau 

```xml
<config version="1.0">
    <Parameters>
        <parameter name="tds2.ACS.MA2.errorUrl" value="https://3ds2.midc.directory.mastercard.com/directory-services/errors"/>
        <parameter name="tds.ThinSrv.ServiceURL" value="10.10.8.69:9401"/>
        <parameter name="tds.HSM.server" value="127.0.0.1:9403"/>
        <!--parameter name="tds.ThreeDSecure.ACS.ACS_URL_BROWSER" value="https://bank.sample.com:8443/way4acs/pa"/-->
        <parameter name="tds.ThreeDSecure.ACS.ACS_URL_BROWSER" value="https://visaotp.seabank.com.vn:443/way4acs2/pa"/>
        <parameter name="tds.localTemplatesHost" value="http://127.0.0.1:8080/"/>
        <parameter name="tds.countryFile" value="country.tbl"/>
        <parameter name="tds.currencyFile" value="currency.tbl"/>
        <parameter name="tds.security.keystore" value="way4acs.ks"/>
        <parameter name="tds.security.keystorePassword" value="DBBF5F953F271E51AC50FBADE95CF8AD0F4810C954"/>
        <parameter name="tds.paTransPoolDir" value="patrans"/>
        <parameter name="tds.PATrans_Maintenance_Frequency" value="5"/>
        <parameter name="tds.PATransTimeOut" value="30000"/>
        <parameter name="tds.maintenaceFrequency" value="30"/>
        <parameter name="tds.amount.spChars" value=" ."/>

        <parameter name="tds.pkcs11.pathToConfig" value="/opt/3dsacs/appserver/applications/way4acs2/webapps/way4acs2/WEB-INF/conf/pkcs11.cfg"/>
        <parameter name="tds.pkcs11.pin" value="B6335D8E156450FE8E39C95A1E9AE4DAFA89D9118072FC0B077C34F77544"/>
        <parameter name="tds.pkcs11.token" value="3D_Token"/>
        <parameter name="tds.pkcs11.keyData.log" value="true"/>

        <!-- Languages we have message files for (see demobank) -->
        <parameter name="tds.languages" value="vn,en"/>
        <!-- Default language, used if "language" parameter is not sent and cannot be autodetected -->
        <parameter name="tds.default.language" value="vn"/>
        <parameter name="tds.use.default.language" value="true"/>
        <parameter name="tds.server.ShowDateFormat" value="dd/MM/YYYY"/>

        <parameter name="tds.tunnels.debug" value="2"/>

        <!-- Cookie-less mode: Them cho KOL 121125 --> 
		<parameter name="tds.url.sessionId" value="jsessionid"/>		
		<parameter name="tds.form-field.sessionId" value="jsessionid"/>

		<!-- Them moi cho KOL 121125 -->
		<parameter name="tds2.ACS.notify.RBA.RReq" value="true"/> 
		<parameter name="tds2.ACS.notify.RBA.ARes" value="true"/>
		<parameter name= "tds.show.loginError" value="false"/> 
        <parameter name= "tds.show.loginFailed" value="false"/>
		
		<parameter name="tds.VA.tunnel.logTlsTrace" value="true"/>
		<parameter name="tds.MA.tunnel.logTlsTrace" value="true"/>
		<parameter name="tds.tunnels.debug" value="2"/>
		<parameter name="tds.pkcs11.limitingOfRequests" value="true"/>
		<parameter name="tds.pkcs11.limitingOfRequests.count" value="30"/>
        <parameter name="tds.pkcs11.limitingOfRequests.timeout" value="5" comment="in secs"/>
		<!-- end -->
		
        <!--DAF-->
        <parameter name="tds2.ACS.Visa.DAF.override.nonY" value="true"/>
        <parameter name="tds2.ACS.VA2.allow3RIWithoutReference" value="true"/>
        <parameter name="tds2.ACS.VA2.save.authenticationResult.inDB" value="true"/>
        <parameter name="tds2.ACS.months.save.authenticationResult.inDB" value="36"/>

        <parameter name="tds2.ISK_EXP_OVERRIDE_DT_FOR_MC" value="yyyyMMdd HH:mm:ss"/>

        <!-- RBA Configuration: -->
        <parameter name="tds2.FrictionlessFlow.RBA.mode" value="IFP"/>
        <parameter name="tds.geoIPData.mmdb.filename" value="GeoLite2-City.mmdb"/>
        <parameter name="tds.fingerprinting.check" value="DB"/>
        <parameter name="tds.fingerprinting.values" value="browser,canvas,connection,cookie,display,flash,fonts,fontsmoothing,formfields,java,language,latency,os,plugins,silverlight,timezone,touch,truebrowser,useragent"/>
        <parameter name="tds2.CORRECT_VND_CURRENCY_EXPONENT" value="true"/>
        <parameter name="tds2.CAVV.MC.mode" value="Spa2Part"/>
        <parameter name="tds2.CAVV.Visa.version" value="V7"/>

        <!-- ***EMV specific (tds2 prefix)*** -->
        <parameter name="tds2.support" value="true"/>
        <parameter name="tds2.strongCheckContentType" value = "false" />
        <parameter name="tds2.VA2.RReq.timeout" value="20000"/>
        <parameter name="tds2.NativeUIFile" value="NativeUI.json"/>

        <parameter name="tds2.VA2.acsOperatorID" value="10067309"/>
        <parameter name="tds2.ACS.BRW.Challenge.URL" value="https://visaotp.seabank.com.vn/way4acs2/brw/challenge"/>
        <parameter name="tds2.ACS.APP.Challenge.URL" value="https://visaotp.seabank.com.vn/way4acs2/app/challenge"/>

        <parameter name="tds2.ACS.ThreeDSMethod.enable" value="true"/>
        <parameter name="tds.Debug.writeTextMessageLog" value="true"/>
        <parameter name="tds.log.AReq" value="true"/>
        <parameter name="tds.log.ARes" value="true"/>
        <parameter name="tds.log.CReq" value="true"/>
        <parameter name="tds.log.CRes" value="true"/>
        <parameter name="tds.log.RReq" value="true"/>
        <parameter name="tds.log.RRes" value="true"/>
        <parameter name="tds.log.Erro" value="true"/>

        <parameter name="tds.log.VEReq" value="true"/>
        <parameter name="tds.log.VERes" value="true"/>
        <parameter name="tds.log.PAReq" value="true"/>
        <parameter name="tds.log.PARes" value="true"/>
        <parameter name="tds.log.Error" value="true"/>
        <parameter name="tds.log.PATransReq" value="true"/>
        <!--parameter name="tds.log.ThinSrv-IssuerList" value="true"/-->

        <!-- Log message processing summary to cards system -->
        <parameter name="tds.log.RemoteEventsLog" value="false"/>

        <!-- Fix for IE/IFrame problem -->
        <!--parameter name="tds.page.header.p3p" value="CP='IDC DSP COR ADM DEVi TAIi PSA PSD IVAi IVDi CONi HIS OUR IND CNT'"/-->
         <parameter name="tds.page.header.p3p" value = "the CP = 'the DSP to IDC the COR the ADM DEVI TAIi the PSA PSD file IVAI IVDi Coni OUR HIS-the IND the CNT'"/>

        <!-- Show page no_auth.jsp in case of "No authentication method" -->
        <parameter name="tds.show.no_auth" value="true"/>

        <!-- Send decline response ('N') in case of "No authentication method" -->
        <parameter name="tds.sendDecline4NoMethod" value="true"/>

        <!-- Connection test (ACS<->Host) messages: -->
        <!--parameter name="tds.connectionTest" value="60" comment="in secs"/-->

        <!-- Switch off connection test messages logging when
         tds.Debug.writeTextMessageLog is true: -->
        <parameter name="tds.log.ThinSrv-ConnectionTest" value="false"/>

        <!-- SSL/TLS tunnels: -->
        <!--parameter name="tds.CH.tunnel" value="8443:127.0.0.1:8080"/>
        <parameter name="tds.CH.tunnel.key" value="#PKCS11:CH_KEY"/>
        <parameter name="tds.CH.tunnel.certs" value="ch_certs.pem"/>
        <parameter name="tds.CH.tunnel.timeout" value="60000"/-->


        <parameter name="tds.VA2.tunnel" value="9702:127.0.0.1:8050"/>
        <parameter name="tds.VA2.tunnel.key" value="#PKCS11:ACS_AREQ_VA22"/>
        <parameter name="tds.VA2.tunnel.certs" value="ACS_AREQ_VA22.pem"/>
        <parameter name="tds.VA2.tunnel.ciphers" value="TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_RSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_128_GCM_SHA256,TLS_RSA_WITH_AES_256_CBC_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256"/>

        <!--parameter name="tds.MA2.tunnel" value="6445:127.0.0.1:8050"/>
            <parameter name="tds.MA2.tunnel.key" value="#PKCS11:ACS_AREQ_MA2"/>
            <parameter name="tds.MA2.tunnel.certs" value="#PKCS11:ACS_AREQ_MA2"/>
        <parameter name="tds.MA2.tunnel.logTlsTrace" value="true"/-->

            <parameter name="tds2.MA2.acsOperatorID" value="ACS-V210-SOUTHEAST-ASIA-CO-75102"/>
            <parameter name="tds2.acsReferenceNumber" value="3DS_LOA_ACS_OPES_020200_00319"/>


        <!-- Cardhoder Pages tunnel: used when demobank port != way4acs port -->
        <!-- Key & certificate must be the same as in CH tunnel above -->
        <!-- parameter name="tds.CP.tunnel" value="2443:127.0.0.1:9090"/>
        <parameter name="tds.CP.tunnel.key" value="#PKCS11:SERVER2"/>
        <parameter name="tds.CP.tunnel.certs" value="server2.pem"/ -->

        <!-- OTP_SMS settings (Example for external SMS delivery service) -->
        <!--parameter name="AS.OTP_SMS.select" value="PHONE=EXT;TEXT=%description% (%amount%);"/>
        <parameter name="AS.OTP_SMS.verify" value="PASSWORD"/-->
                <parameter name="AS.OTP_SMS.select" value="PHONE=*;TEMPLATE:=sms_text.jsp;"/>
                <parameter name="AS.OTP_SMS.service" value="PHONE=*;TEMPLATE:=sms_text.jsp;"/>
                <parameter name="AS.OTP_SMS.verify" value="PASSWORD,TRIES,PSW_RENEW"/>
                <parameter name="AS.OTP_SMS.selectPhone.list" value="PHONELIST"/>
                <parameter name="AS.OTP_SMS.selectPhone.list.PhoneHideMask" value="0:4"/>


                <!--OTP Email Setting-->
                <parameter name="AS.OTP_EMAIL.select" value="PHONE=*;TEMPLATE:=email_text.jsp;"/>
                <parameter name="AS.OTP_EMAIL.service" value="PHONE=*;TEMPLATE:=email_text.jsp;"/>
                <parameter name="AS.OTP_EMAIL.verify" value="PASSWORD,TRIES,PSW_RENEW"/>
                <parameter name="AS.OTP_EMAIL.selectPhone.list" value="PHONELIST"/>
                <parameter name="AS.OTP_EMAIL.selectPhone.list.PhoneHideMask" value="3:0"/>
                <parameter name="tds.maxLoginTries" value="3"/>

        <!-- OTP_LIST settings example -->
        <!--parameter name="AS.OTP_LIST.verify" value="PASSWORD,TRIES"/>
        <parameter name="AS.OTP_LIST.select" value=""/-->

        <!-- OTP_CAP settings example (OTP_DPA is similar) -->
        <!--parameter name="AS.OTP_CAP.verify" value="PASSWORD"/>
        <parameter name="AS.OTP_CAP.select" value=""/-->

        <!-- Enrollment servlet settings -->
        <!--parameter name="tds.enroll.pagesURL" value="https://bank.sample.com:8443/demobank/enroll/"/>
        <parameter name="tds.enroll.timeout" value="600"/>
        <parameter name="tds.enroll.cache" value="true"/>
        <parameter name="tds.enroll.reEnrollment" value="true"/-->

        <!-- Health monitoring compatibility mode: -->
        <!--parameter name="tds.HM.mode" value="1"/-->

        <!-- Additional settings for PING Command -->
        <!--parameter name="tds.ping.demobank.pageURL" value="https://3ds-rhel62.spb.openwaygroup.com:8443/demobank/common/ping.jsp"/-->

        <!-- Settings for emergency mode -->
        <!--parameter name="tds.emergency_mode.state" value="false"/>
        <parameter name="tds.emergency_mode.date_format" value="dd.MM.yyyy HH:mm"/>
        <parameter name="tds.emergency_mode.start_work" value="02.10.2014 15:55"/-->

        <!-- Settings for install pkcs11 -->
        <parameter name="tds.InvalidActionRequest.N.status" value="true"/>
        <parameter name="tds.invalidActionMode" value="1"/>
    </Parameters>
</config>
```

---
### ACS2: tăng 16G
- Sửa tham số heap java **`/opt/3dsacs/appserver/applications/way4acs2/conf/config.properties`**
```
jvm_params=-Xmx16g -Xms16g
```

---------------
### EGW
- Sửa tham số heap java ở file **`/opt/3dsegw/appserver/applications/TransSwitch/conf/config.properties`**
```
jvm_params=-Xmx16g -Xms8g
```
- Vào thư mục **`/opt/3dsegw/appserver/applications/EgateWay/app/datagate/scripts`**
- sửa file **`rba_analyze.lua`** bỏ phần check file `client.txt`, các nội dung sau được cmt
 ```lua
     -- Doc du lieu tu file client.txt,neu tim thay thi chi tra ve duy nhat thong tin cua the do
-- begin KOL 121125
--    local clientArray = readCardOnly("/opt/3dsegw/appserver/applications/EgateWay/app/datagate/scripts/data/client.txt", areqCardBase64, nil, false)
--    if clientArray then
--        local clientPhone = clientArray[1][3]
--        local clientEmail = clientArray[1][4]
--        local clientShortName = clientArray[1][2]
--              if (clientShortName == nil or clientShortName =='' or clientEmail == nil or clientEmail =='' or clientPhone ==nil or clientPhone =='') then
--                  ext:log("client information in system is missing")
--                      return false
--              end
--        --ext:log("client card base64-----" .. clientArray[1][1])
--        --ext:log("client Phone-----" .. clientPhone)
--        --ext:log("client Email--" .. clientEmail)
--        --ext:log("areqEmail--" .. areqEmail)
--              if (compareString(clientPhone, areqPhone)) then
--            ext:log("phone matched")
--            if (compareString(clientEmail,areqEmail)) then
--                ext:log("email matched")
--                if (compareString(clientShortName, areqCardholerN)) then
--                    ext:log("Client phone,email,shortname matched")
--                    return true
--                end
--            end
--        end
--    end
-- end KOL 121125
 ```
 ---
### Thực hiện backup log ở  TS của EGW
 - Từ `/opt/3dsegw/appserver/applications/TransSwitch/webapps/TransSwitch/WEB-INF/logs` ra `/backupfs`
 - Nội dung file script **`/home/way4/scripts/backup_ts_extlog.sh`** chạy 1h 1 lần 
 
 ```sh
 #!/bin/sh
# ==========================================================
# Script: sync_ext_logs.sh
# Mô tả : Đồng bộ các file ext_*.log từ SRC_DIR sang DES_DIR
# ==========================================================

set +x
SRC_DIR=/opt/3dsegw/appserver/applications/TransSwitch/webapps/TransSwitch/WEB-INF/logs
DES_DIR=/backupfs/${hostname}/logs/TransSwitch
mkdir -p $DES_DIR

# Thực thi đồng bộ
find "$SRC_DIR" -type f -name 'ext_*.log' -mmin -65 -print | while IFS= read -r file; do
    echo "➡️  Copy: $file"
    cp -p "$file" "$DES_DIR"/ || echo "⚠️  Lỗi copy file: $file"
done

echo "✅ Hoàn tất!"
 ```
 ---
### Thay đổi sysctl -p của  OS
- Tạo file tunning hệ thống
```sh
touch /etc/sysctl.d/99-tuning.conf
vi /etc/sysctl.d/99-tuning.conf
```
- Nội dung:
```properties
# --- 3DS high-load tuning ---
net.core.somaxconn = 16384
net.core.netdev_max_backlog = 65536
net.ipv4.tcp_max_syn_backlog = 16384
# Cổng tạm thời & timeout cho nhiều kết nối ngắn
net.ipv4.ip_local_port_range = 10240 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
# TCP buffers (tùy đường truyền/RTT, có thể giảm nếu memory hạn hẹp)
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
# Hàng đợi QoS mặc định
# net.core.default_qdisc = fq_codel
# Số file mở toàn hệ thống (tăng nếu workload lớn)
fs.file-max = 2097152 
```
- Áp dụng
```sh
sysctl --system
```

- ***Chú thích:*** 
 **1. Hàng đợi & backlog cho kết nối cao tải**

| Tham số                                | Ý nghĩa                                                                                                  |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `net.core.somaxconn = 16384`           | Tăng tối đa hàng đợi `accept()` của ứng dụng → tránh nghẽn khi có quá nhiều kết nối chờ server xử lý.    |
| `net.core.netdev_max_backlog = 65536`  | Tăng hàng đợi packet ở NIC khi kernel chưa kịp xử lý → chống mất gói khi burst traffic lớn.              |
| `net.ipv4.tcp_max_syn_backlog = 16384` | Tăng hàng chờ **SYN** (half-open) khi nhiều client cùng bắt tay kết nối → giảm SYN drop trong high load. |
**2. Tối ưu cho lượng lớn kết nối ngắn (3DS / API burst)**

| Tham số                                      | Ý nghĩa                                                                                |
| -------------------------------------------- | -------------------------------------------------------------------------------------- |
| `net.ipv4.ip_local_port_range = 10240 65535` | Mở rộng dải port tạm để server tạo nhiều outbound connection mà không bị hết port.     |
| `net.ipv4.tcp_fin_timeout = 15`              | Giảm thời gian giữ socket FIN_WAIT2 → giải phóng tài nguyên nhanh.                     |
| `net.ipv4.tcp_tw_reuse = 1`                  | Cho phép kernel tái sử dụng TIME_WAIT → giảm nghẽn khi có nhiều kết nối ngắn liên tục. |
**3. Tăng buffer TCP để chịu băng thông/RTT lớn**

| Tham số                                    | Ý nghĩa                                                                              |
| ------------------------------------------ | ------------------------------------------------------------------------------------ |
| `net.core.rmem_max = 134217728`            | Max receive buffer cho socket → phù hợp đường truyền tốc độ cao.                     |
| `net.core.wmem_max = 134217728`            | Max send buffer → giúp đẩy dữ liệu nhanh, tránh nghẽn.                               |
| `net.ipv4.tcp_rmem = 4096 87380 134217728` | Min/Default/Max buffer nhận của TCP → cho phép kernel scale theo băng thông thực tế. |
| `net.ipv4.tcp_wmem = 4096 65536 134217728` | Min/Default/Max buffer gửi → tối ưu khi throughput cao hoặc RTT lớn.                 |
**4. Hàng đợi QoS (optional)**

| Tham số                             | Ý nghĩa                                                                    |
| ----------------------------------- | -------------------------------------------------------------------------- |
| `net.core.default_qdisc = fq_codel` | (Không bật) Cơ chế chống bufferbloat thông minh, ổn định latency dưới tải. |
|                                     |                                                                            |
**5. File descriptor**

| Tham số                 | Ý nghĩa                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------- |
| `fs.file-max = 2097152` | Tăng số lượng file/socket kernel có thể mở hệ thống-wide → phù hợp server high-concurrency. |
 **Tóm tắt**
* **Backlog lớn** → không drop SYN/accept khi traffic đột biến.
* **Port range rộng + TW reuse** → phục vụ nhiều kết nối ngắn-lặp.
* **TCP buffers to the max** → tận dụng băng thông cao.
* **file-max lớn** → chống “too many open files” khi burst.

- [x] hardening nốt nhánh 2 master ✅ 2025-12-25
- [x] Chuyển hết về nhánh 2 ✅ 2025-11-14
- [x] Cấu hình visa acs 01 về visa egw01 ✅ 2025-11-14
- [x] restart safenet 33 của nhánh 1 ✅ 2025-11-17
- [x] chuyển visa về nhánh 1 ✅ 2025-11-17