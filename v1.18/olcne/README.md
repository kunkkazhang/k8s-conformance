The latest prerequisite and software installation steps are documented in the [Getting Started](https://docs.oracle.com/en/operating-systems/olcne/start/) guide. 
This is a site-specific example to validate conformance and not to be used in production. Users should review the _Getting Started_ guide to understand the reasoning behind the choices made, especially in relation to security. 
***
Oracle Linux Cloud Native Environment uses three open source tools to simplify the installation and management of Kubernetes and other CNCF projects as objects called modules.
These include: Platform API Server `olcne-api-server`, Platform Agent `olcne-agent` and Platform Command-Line Interface `olcnectl`

This deployment uses six Oracle Linux 8 nodes; one operator node, three control plane nodes and two worker nodes.
It is expected that an administrator user opc is present on each node.
1. (On all nodes) Enable necessary dnf repositories
	~~~
	sudo dnf install -y oracle-olcne-release-el8
	sudo dnf config-manager --enable ol8_olcne_12 ol8_addons ol8_baseos_latest ol8_UEKR6
	~~~
2. Install RPMS
	- On operator node
		`sudo dnf install -y olcnectl olcne-api-server olcne-utils`
	- On control plane and worker nodes
		`sudo dnf install -y olcne-agent olcne-utils`
3. Configure Firewalls
	- On operator node
	~~~
	sudo firewall-cmd --add-port=8091/tcp --permanent
	sudo firewall-cmd --reload
	~~~
	- On control plane nodes
	~~~
	sudo firewall-cmd --add-interface=cni0 --zone=trusted --permanent 
	sudo firewall-cmd --add-port=8090/tcp --permanent
	sudo firewall-cmd --add-port=10250/tcp --permanent
	sudo firewall-cmd --add-port=10255/tcp --permanent
	sudo firewall-cmd --add-port=8472/udp --permanent
	sudo firewall-cmd --add-port=10252/tcp --permanent
	sudo firewall-cmd --add-port=10251/tcp --permanent
	sudo firewall-cmd --add-port=2379/tcp --permanent
	sudo firewall-cmd --add-port=2380/tcp --permanent
	sudo firewall-cmd --add-port=6443/tcp --permanent
	sudo firewall-cmd --add-port=6444/tcp --permanent
	sudo firewall-cmd --add-protocol=vrrp --permanent
	sudo firewall-cmd --reload
	~~~
	- On worker nodes
	~~~
	sudo firewall-cmd --add-interface=cni0 --zone=trusted --permanent 
	sudo firewall-cmd --add-port=8090/tcp --permanent
	sudo firewall-cmd --add-port=10250/tcp --permanent
	sudo firewall-cmd --add-port=10255/tcp --permanent
	sudo firewall-cmd --add-port=8472/udp --permanent
	sudo firewall-cmd --reload
	~~~
4. Configure and start services
	- On operator node
	~~~
	sudo systemctl enable --now olcne-api-server.service
	~~~
5. Configure example X.509 certificates
	- On the operator node
	~~~
	cd /etc/olcne
	sudo ./gen-certs-helper.sh \
	   --cert-request-organization-unit "My Company Unit" \
	   --cert-request-organization "My Company" \
	   --cert-request-locality "My Town" \
	   --cert-request-state "My State" \
	   --cert-request-country US \
	   --cert-request-common-name cloud.example.com \
	   --nodes operator.example.com,master1.example.com,master2.example.com,master3.example.com,worker1.example.com,worker2.example.com
	~~~
6. Distribute example X.509 certificates
	- On the operator node
	~~~
	ssh-keygen -N '' -f ~/.ssh/id_rsa
	for HOST in operator.example.com master1.example.com master2.example.com master3.example.com \
	            worker1.example.com worker2.example.com
	do
        ssh-copy-id ${HOST}
	done
	bash -ex /etc/olcne/configs/certificates/olcne-tranfer-certs.sh
	~~~
7. Initialize `olcne-api-server` and `olcne-agent` services
	- On operator node
		`sudo /etc/olcne/bootstrap-olcne.sh --secret-manager-type file --olcne-node-cert-path /etc/olcne/configs/certificates/production/node.cert --olcne-ca-path /etc/olcne/configs/certificates/production/ca.cert --olcne-node-key-path /etc/olcne/configs/certificates/production/node.key --olcne-component api-server`
	- On control plane nodes
		`sudo /etc/olcne/bootstrap-olcne.sh --secret-manager-type file --olcne-node-cert-path /etc/olcne/configs/certificates/production/node.cert --olcne-ca-path /etc/olcne/configs/certificates/production/ca.cert --olcne-node-key-path /etc/olcne/configs/certificates/production/node.key --olcne-component agent`
	- On worker nodes
		`sudo /etc/olcne/bootstrap-olcne.sh --secret-manager-type file --olcne-node-cert-path /etc/olcne/configs/certificates/production/node.cert --olcne-ca-path /etc/olcne/configs/certificates/production/ca.cert --olcne-node-key-path /etc/olcne/configs/certificates/production/node.key --olcne-component agent`
8. Define site-specific variables
	- On the operator node
	~~~
	export certificate_dir="/etc/olcne/configs/certificates/production/"
	export api_server=operator.example.com:8091
	export master_addr=master1.example.com:8090,master2.example.com:8090,master3.example.com:8090
	export worker_addr=worker1.example.com:8090,worker2.example.com:8090
	export lb_ip=100.100.231.203
	~~~
9. Create an environment, an instance of a Oracle Linux Cloud Native Environment deployment
	- On the operator node
	~~~
	olcnectl \
	    --api-server $api_server \
	    --olcne-ca-path ${certificate_dir}/ca.cert \
	    --olcne-node-cert-path ${certificate_dir}/node.cert \
	    --olcne-node-key-path ${certificate_dir}/node.key \
	    --update-config \
	    environment create \
	        --environment-name test-environment
	~~~
10. Create the Kubernetes module definition within that environment
	- On the operator node
	~~~
	olcnectl module create \
	    --environment-name test-environment \
	    --module kubernetes \
	    --name test-module-kube \
	    --container-registry container-registry.oracle.com/olcne \
	    --master-nodes ${master_addr} \
	    --worker-nodes ${worker_addr} \
	    --virtual-ip ${lb_ip} \
		--selinux enforcing
	~~~
11. Validate and deploy the Kubernetes module within that environment
	- On the operator node
	~~~
	olcnectl --environment-name test-environment \
	    --name test-module-kube \
	    module validate
	olcnectl --environment-name test-environment \
	    --name test-module-kube \
	    module install
	~~~
12. Verify Kubernetes state before executing conformance test
	- On the operator node
	Configure `kubectl`
	~~~
	mkdir "$HOME/.kube"
	olcnectl module property get \
        --environment-name test-environment \
        --name test-module-kube \
        --property kubecfg | base64 -d > "$HOME/.kube/config"
	kubectl get nodes
	NAME                    STATUS   ROLES    AGE     VERSION
	master1.example.com     Ready    master   4h33m   v1.18.10+2.el8
	master2.example.com     Ready    master   4h31m   v1.18.10+2.el8
	master3.example.com     Ready    master   4h32m   v1.18.10+2.el8
	worker1.example.com     Ready    <none>   4h30m   v1.18.10+2.el8
	worker2.example.com     Ready    <none>   4h30m   v1.18.10+2.el8
	~~~
	Obtain sonobuoy
	~~~
	export SONOBUOY_VERSION="0.18.5"
	export SONOBUOY_FILENAME="sonobuoy_${SONOBUOY_VERSION}_linux_amd64.tar.gz"
	curl -LO https://github.com/vmware-tanzu/sonobuoy/releases/download/v${SONOBUOY_VERSION}/${SONOBUOY_FILENAME}
	tar zxf ${SONOBUOY_FILENAME}
	~~~
	Begin test
	~~~
	./sonobuoy version --kubeconfig ~/.kube/config
	Sonobuoy Version: v0.18.5
	MinimumKubeVersion: 1.16.0
	MaximumKubeVersion: 1.18.99
	GitSHA: a79a43a99680f7274e6315dbf09fadc8fd81a6e0
	API Version:  v1.18.10+2.el8
	./sonobuoy run --mode=certified-conformance
	~~~
