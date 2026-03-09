## Import cert vào os 
cp MC_CA.pem /etc/pki/ca-trust/source/anchors/
cp MC_ROOT.pem /etc/pki/ca-trust/source/anchors/
update-ca-trust extract



# Import cert vào file 

keytool -delete -alias MC_CA -trustcacerts -v -keystore /opt/3dsacs/appserver/applications/way4acs2/webapps/way4acs2/WEB-INF/conf/way4acs.ks -storepass changeit

keytool -list -alias MC_CA -trustcacerts -v -keystore /opt/3dsacs/appserver/applications/way4acs2/webapps/way4acs2/WEB-INF/conf/way4acs.ks -storepass changei


keytool -import -alias MC_CA -trustcacerts -v -keystore /opt/3dsacs/appserver/applications/way4acs2/webapps/way4acs2/WEB-INF/conf/way4acs.ks -storetype JCEKS -file MC_CA.pem


## Thực hiện Import ACS_AREQ_VA22

```sh
ctkmu d -s0 -nACS_AREQ_VA22
ctcert i -s0 -lACS_AREQ_VA22 -fACS_AREQ_VA22.pem

ctkmu d -s0 -nACS_AREQ_VA22
ctcert i -s0 -lACS_AREQ_VA22 -fACS_AREQ_VA22.pem
```


Plan:

Hiện tại hệ thống 3Dsecure MasterCard đang chạy 2 node.

Name: dc1-3dsacs01-visa
Address: 10.9.218.68
>= node này đang chạy chính

Name: dc1-3dsacs02-visa
Address: 10.9.218.69
>= node này làm dự phòng.

Theo Plan sẽ thực hiện trên node dự phòng trước là 3dsacs04.
B1: Snapshot lại server 
B2: thực hiện thay cert theo check list đính kèm từng nhánh.
B3: sau khi thực hiện xong báo lại OM, đêm cùng ngày OM sẽ thực hiện chuyển server nhánh, nếu ok thì để hệ thống chạy trên nhánh này, theo dõi sự ổn định và thực hiện tiếp trên nhánh  còn lại.

nếu không đạt chuyển về nhánh cũ, đồng thời QTHT restore lại bản backup snapshot trước khi thực hiện. 