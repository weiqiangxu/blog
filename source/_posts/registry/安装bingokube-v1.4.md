安装bingokube-v1.4.2
	single-node.tar.gz 安装包解压 /bingokube/single-node
	安装docker 
		sh install-docker.sh
	离线安装(ip地址是节点管理网卡地址)
		sh install.sh 192.168.1.1 offline
	访问管理后台
		http://192.168.1.1:8989
	添加集群
	迁移集群
		系统配置 -> 部署设置 -> 迁移
			选集群和填写新kubeverse访问的ip （IP地址）
		http://10.16.203.134:29788/
		安装ingress-nginx
			hostnetwork + spec.ingressCLassName
		docker stop $(docker ps -aq)
			停止单机的etcd\nginx\registry等