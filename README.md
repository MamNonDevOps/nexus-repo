# nexus-repo
![alt text](https://github.com/MamNonDevOps/nexus-repo/blob/main/images/overview_x1.png)

## User-data
```
#!/bin/bash -ex
yum -y update
yum -y install docker git
usermod -aG docker ec2-user
systemctl enable docker
systemctl start docker
```

## EC2 Nexus repo
t2.medium (2 vCPU, 4 GB Ram), recommend 4 vCPU trở lên
EBS: 20 GB

```
mkdir /some/dir/nexus-data
sudo chown -R 200 /some/dir/nexus-data
docker run -d -p 8081:8081 --name nexus -v /some/dir/nexus-data:/nexus-data sonatype/nexus3
```
Mở thêm port và sử dụng đường dẫn tuyệt đối $(pwd)
```
docker run -d -p 8081:8081 -p 8082:8082 --name nexus -v $(pwd)/nexus-data:/nexus-data sonatype/nexus3
```

Kiểm tra log container `nexus` xem đã hoàn thành cài đặt hay chưa `docker log -f nexus`

---
### SSH tunel từ localhost
```
ssh -i key-pair.pem -L 8081:10.0.0.1:8081 ec2-user@54.0.0.1
```
Từ localhost, truy cập vào web `locahost:8081` để mở giao diện quản trị

Tài khoản đăng nhập mặc định là `admin`, password xem tại file /nexus-data/admin.password

---

Tạo Blod store cho từng Repo

Docker-hosted
- pull các image đang có trên repo
- push các image đang có trên máy host

Doker-proxy
- pull các image trên repo, nếu repo không có thì lấy image từ remote repository khác (Docker hub)
- ?? push

Docker-group, thêm Docker-hosted và Doker-proxy
- pull các image trên repo, nếu repo không có thì lấy image từ remote repository khác (Docker hub) (giống Docker proxy)
- ?? push

---

Tạo role `nx-cicd`, có các privilege:
* nx-healthcheck-read
* nx-search-read
* nx-repository-view-*-*-read
* nx-repository-view-*-*-browse
* nx-repository-view-*-*-edit
* nx-repository-view-*-*-add

Tạo local user `nx-cicd` với role trên

---

Actice thêm **Docker Bearer Token Realm** trong cài đặt Security -> Realms

Allow anonymous user to acces the server trong cài đặt Security -> Anonymous Access

## EC2 CICD

command login: (sử dụng user `nx-cicd`)
```
docker login -u nx-cicd 10.0.0.1:8082
```

Với `10.0.0.1` là private ip của EC2 Nexus repo

command tag:
```
docker tag image-name:tag 10.0.0.1:8082/image-name:tag
```
command push image:
```
docker push 10.0.0.1:8082/image-name:tag
```

---
Error response from daemon: Get "https://10.0.0.1:8082/v2/": http: server gave HTTP response to HTTPS client
```
sudo su
touch /etc/docker/daemon.json
echo "{ \"insecure-registries\":[\"172.31.19.205:8082\"] }" >> /etc/docker/daemon.json
systemctl restart docker
su ec2-user
```

## EC2 private
Các server pull image về cần tạo user, role riêng
Role `nx-ec2`, có các privilege:
* nx-healthcheck-read
* nx-search-read
* nx-repository-view-*-*-read
* nx-repository-view-*-*-browse

command pull:
```
docker pull 10.0.0.1:8082/image-name:tag
```

## Tối ưu
EC2 Nexus repo chỉ cần hoạt động khi cần push hoặc pull
=> nên để tiết kiệm thì chỉ cần Start khi cần dùng, Stop sau khi dùng xong
=> ?? địa chỉ public ip bị thay đổi sau mỗi lần restart

=> gộp chung EC2 Nexus repo và EC2 CICD, dùng Elastic IP
