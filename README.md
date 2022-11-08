# nexus-repo
![alt text](https://github.com/MamNonDevOps/nexus-repo/blob/main/images/overview_x2.png)

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

## EC2 CICD
local user-name: `nx-cicd`

role: `nx-cicd`

privilege:
* nx-healthcheck-read
* nx-search-read
* nx-repository-view-*-*-read
* nx-repository-view-*-*-browse
* nx-repository-view-*-*-edit
* nx-repository-view-*-*-add

command login:
```
docker login -u nx-cicd 54.174.173.12:8082
```
với giả định `54.174.173.12` là public ip của EC2 Nexus repo, port 8082 đã mở để thao tác push/pull

command tag:
```
docker tag image-name:tag new-image-name:new-tag
```
command push image:
```
docker push 54.174.173.12:8082/new-image-name:new-tag
```

---
Error response from daemon: Get "https://54.174.173.12:8082/v2/": http: server gave HTTP response to HTTPS client
```
sudo su
touch /etc/docker/daemon.json
echo "{ \"insecure-registries\":[\"54.174.173.12:8082\"] }" >> /etc/docker/daemon.json
systemctl restart docker
su ec2-user
```

## EC2 private
local user-name: `nx-ec2`

role: `nx-ec2`

privilege:
* nx-healthcheck-read
* nx-search-read
* nx-repository-view-*-*-read
* nx-repository-view-*-*-browse

command login:
```
docker login -u nx-ec2 54.174.173.12:8082
```
command pull:
```
docker pull 54.174.173.12:8082/new-image-name:new-tag
```

## Tối ưu
EC2 Nexus repo chỉ cần hoạt động khi cần push hoặc pull
=> nên để tiết kiệm thì chỉ cần Start khi cần dùng, Stop sau khi dùng xong.
=> Nhưng như vậy sẽ địa chỉ public ip bị thay đổi sau mỗi lần restart

=> gộp chung EC2 Nexus repo và EC2 CICD, dùng Elastic IP
