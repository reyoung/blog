## Step.1 在MacOS上安装 K8S

可以使用vagrant进行部署。vagrant是一个虚拟机管理工具，类似于docker，只不过管理的东西是虚拟机而已。

TIPS:

* MacOS上部署K8S需要的vagrant版本是`1.8.6`，最新版1.8.7有bug。
* 同时，vagrant默认自带的openssl版本过老，需要使用brew里面的openssl替换一下。命令 `sudo ln -sf /usr/local/bin/openssl /opt/vagrant/embedded/bin/openssl`

使用 `git clone https://github.com/coreos/coreos-kubernetes.git` 下载coreos的k8s配置。cd进`coreos-kubernetes/multi-node/vagrant/`修改一下config.rb即可以使用k8s集群了。

* 打开虚拟机 `vagrant up`
* 打开虚拟机后，k8s的安装还需要几分钟。如果想要查看进展。就`vagrant ssh c1 -- -A` 登入control机器，然后journalctl来看一下日志。
* 在相同命令运行`kubectl config use-context vagrant-multi`配置k8s的访问
* kubectl get nodes查看当前节点数。

