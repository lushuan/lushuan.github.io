# kubelet目录满问题记录

## 现象
k8s 某台主机的/var/lib/kubelet目录存储使用率超过80% 使用了40多G存储

## 分析
1. 申请root权限  
2. cd /var/lib/kubelet
执行du -h -d 1
发现 pod目录占据大量存储

3. cd pod  
继续执行du -h -d 1  
定位到pod  
2753dcf9-4113-4e65-a944-278bcbd8d6d8  

4. 按上面方法继续执行  
发现最终路径为  
/var/lib/kubelet/pods/2753dcf9-4113-4e65-a944-278bcbd8d6d8/volumes/kubernetes.io~empty-dir/storage-volume

5. docker ps|grep 2753dcf9-4113-4e65-a944-278bcbd8d6d8  
定位容器为prometheus  

6. docker inspect 122ec2fbbd1e  
![kubelet](/images/kubernetes/problem_and_solve/kubelet.png)
该容器使用了empty-dir挂载内部的/data



