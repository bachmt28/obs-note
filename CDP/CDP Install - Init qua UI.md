## Init base add cluster 
![[Pasted image 20260109110853.png]]
![[Pasted image 20260109110910.png]]
![[Pasted image 20260109110955.png]]
![[Pasted image 20260109111638.png]]

### Init Service 
- Add Cluster - Configuration  -> CustomService -> ZooKeeper
![[Pasted image 20260109111830.png]]
- Set dababase rman
-![[Pasted image 20260109112026.png]]
- Enter Required Parameters Skip
- Nếu không có gì thay đổi, giữ như mặc định. Chú ý phân vùng /var/lib/zookeeper đảm bảo đủ dung lượng 
![[Pasted image 20260109112504.png]]
- Init zookeepper lỗi do thiếu java home 
	- set qua host configuration và restart lại 
	- như hình:
	- ![[Pasted image 20260109112838.png]]
	- Restart zookeper instnace và đợi 1 lúc cho zoo cannary
### Add Service HDFS
![[Pasted image 20260109114246.png]]![[Pasted image 20260109114320.png]]
![[Pasted image 20260109114409.png]]- Nếu không quy hoạch phân vùng dfs riêng thì để mặc định như hình . Chú ý đảm bảo dung lượng
![[Pasted image 20260109114528.png]]
- Finish
### Add Service YARN
![[Pasted image 20260109115553.png]]- default
![[Pasted image 20260109115616.png]]![[Pasted image 20260109115757.png]]

### Add Service HIVE
![[Pasted image 20260109120156.png]]
![[Pasted image 20260109120245.png]]
![[Pasted image 20260109120424.png]]
### Add Service ICEBERGE
Tự có sau khi cài HIVE
Nhớ start lên 
### Add Service SPARK 3
![[Pasted image 20260109121328.png]]
default
![[Pasted image 20260109121351.png]]
![[Pasted image 20260109121516.png]]
- Start
### Add Service IMPALA
![[Pasted image 20260109121802.png]]![[Pasted image 20260109121907.png]]
![[Pasted image 20260109122035.png]]![[Pasted image 20260109123706.png]]