BÁO CÁO TRIỂN KHAI WEBCLUSTER
-------------------------------------------------------------------------------------------------------------------------------

***Mô hình tổng thể gồm 5 Node trong đó:**

- 3 nodes backend triển khai wordpress + database Mysql
- 2 nodes đảm nhiệm chức năng HighAvaibility + Loadblance + KeepAlive

**Phương án triển khai**
- 3 nodes backend sử dụng Percona Xtrabackup để đồng bộ dữ liệu
- 

