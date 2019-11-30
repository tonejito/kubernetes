SHELL=/bin/bash

WORKDIR=/vagrant
VERSION=1.15.1
KUBE_VERSION=${VERSION}-00
POD_CIDR=172.31.0.0/16
K8S_MASTER_IP=192.168.10.12
K8S_MASTER_NAME=k8s-master
K8S_WORKER_IP=192.168.10.13
K8S_WORKER_NAME=k8s-worker
K8S_TOKEN=xxxxxxxxxxxxxxxxxxxxxxx
K8S_DISCOVERY_TOKEN_CA_CERT_HASH=sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

all:
	echo 'execute with either "master" or "worker"'

clean:
	vagrant destroy -f

again:	clean reset

reset:
	vagrant up --no-provision
	vagrant halt
	vagrant snapshot push
	vagrant up --provision
	vagrant halt
	vagrant snapshot push
	vagrant up

pop:
	vagrant snapshot pop
	vagrant snapshot pop

common:
	ufw disable
	apt -qqy install httping
	echo "${K8S_MASTER_IP}	${K8S_MASTER_NAME}" | tee -a /etc/hosts
	grep "${K8S_MASTER_NAME}" /etc/hosts
	apt -qq update
	apt -qqy install docker.io
	echo "deb  http://apt.kubernetes.io/  kubernetes-xenial  main" | tee /etc/apt/sources.list.d/kubernetes.list
	curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
	apt -qq update
	apt -qqy install kubeadm=${KUBE_VERSION} kubelet=${KUBE_VERSION} kubectl=${KUBE_VERSION}
	kubeadm config images pull

provision_users:
	test -d /root/.kube || \
	  mkdir -vp /root/.kube ; \
	cp -v /etc/kubernetes/admin.conf /root/.kube/config ; \
	echo 'source <(kubectl completion bash)' >> /root/.bashrc
	
	for USR in vagrant ubuntu ; \
	do \
	  adduser $${USR} docker ; \
	  test -d /home/$${USR}/.kube || \
	    mkdir -vp /home/$${USR}/.kube ; \
	  cp -v /etc/kubernetes/admin.conf /home/$${USR}/.kube/config ; \
	  echo 'source <(kubectl completion bash)' >> /home/$${USR}/.bashrc ; \
	  chown -cR $${USR}:$${USR} /home/$${USR} ; \
	done ;

master:	common
	sed -i -e 's/$(shell hostname -s)/${K8S_MASTER_NAME}/' /etc/hosts
	grep "${K8S_MASTER_NAME}" /etc/hosts
	#kubeadm init --config=kubeadm-config.yaml --upload-certs 2>&1 | tee kubeadm-init.out
	kubeadm init --kubernetes-version=${VERSION} --pod-network-cidr=${POD_CIDR} --apiserver-advertise-address=${K8S_MASTER_IP}  --upload-certs 2>&1 | tee kubeadm-init.out
	cp -v /etc/kubernetes/admin.conf kubeconfig.yml
	$(MAKE) hash provision_users
	$(MAKE) remove-taint
	kubectl apply -f rbac-kdd.yaml
	kubectl apply -f calico.yaml
	kubectl get nodes
	kubectl describe nodes

remove-taint:
	-kubectl taint nodes --all node-role.kubernetes.io/master-

hash:
	openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
	openssl rsa -pubin -outform der 2>/dev/null | \
	openssl dgst -sha256 -hex | \
	sed 's/^.* //' | \
	tee k8s-cert-hash

worker:	common
	ping -c 1 ${K8S_MASTER_NAME}
	httping -c 1 https://${K8S_MASTER_NAME}:6443/
	test -d /etc/kubernetes || mkdir -vp /etc/kubernetes
	cp -v kubeconfig.yml /etc/kubernetes/admin.conf
	$(MAKE) provision_users
	kubectl get nodes
	kubectl describe nodes
	$(MAKE) join

join:
	cp -v kubelet-default /etc/default/kubelet
	echo 'KUBELET_EXTRA_ARGS="--node-ip=${K8S_WORKER_IP}"' | tee -a /etc/default/kubelet
	service kubelet restart
	kubeadm join ${K8S_MASTER_NAME}:6443 \
	  --node-name "${K8S_WORKER_NAME}-$(shell echo '${K8S_WORKER_IP}' | tr '.' '-')" \
	  --token ${K8S_TOKEN} \
	  --discovery-token-ca-cert-hash ${K8S_DISCOVERY_TOKEN_CA_CERT_HASH} 
	kubectl get nodes
	kubectl describe node "${K8S_WORKER_NAME}-$(shell echo '${K8S_WORKER_IP}' | tr '.' '-')"
	kubectl get pods --all-namespaces -o wide

patch:
	kubectl patch ds calico-node --namespace kube-system --type json --patch '$(shell cat calico.patch)'

