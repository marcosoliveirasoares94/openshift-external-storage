#
# Install NFS Server
#

# Create a Centos/RHEL 7 based instance and run the following commands as root

	yum install -y nfs-utils
	systemctl enable rpcbind
	systemctl enable nfs-server
	systemctl start rpcbind
	systemctl start nfs-server

	mkdir /ifs/storage && chmod -R 755 /ifs/storage

	firewall-cmd --permanent --zone=public --add-service=nfs
	firewall-cmd --permanent --zone=public --add-service=rpcbind
	firewall-cmd --reload

# To allow all clients put *
	vi /etc/exports
	/ifs/storage *(rw,sync,no_root_squash)

or you can limit to a subnet/IP
	/ifs/storage 172.16.2.0/24(rw,sync,no_root_squash)
	systemctl restart nfs-server

#
# Install nfs-utils package on all openshift nodes
#
	yum install -y nfs-utils

#
# Download kubernetes-incubator
#
	curl -L -o kubernetes-incubator.zip https://github.com/marcosoliveirasoares94/openshift-external-storage/archive/main.zip

	unzip kubernetes-incubator.zip && mv openshift-external-storage-main openshift-external-storage && rm -rf openshift-external-storage-main

	cd openshift-external-storage/nfs-client/

#
# Create Workspace
#
oc new-project managed-nfs-storage --display-name="managed-nfs-storage" --description="Managed NFS Storage"

# Change default namespace with current project/namespace. If you are in different project right now. Please switch to target project before running the commands below:

	NAMESPACE=`oc project -q`
	sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml
    sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/deployment.yaml

# Create the service account and role bindings.
	oc create -f deploy/rbac.yaml

# Allow run as root and mount permission to nfs-client-provisioner. To grant this permission, you must be logged in to openshift with admin privileges.
	oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:$NAMESPACE:nfs-client-provisioner

# edit deploy/deployment.yaml
# Change the following parameters with yours <YOUR NFS SERVER HOSTNAME> It is 172.16.2.5 in my case and change /var/nfs with /var/nfsshare
# which we configured above. Also You don’t have to change PROVISIONER_NAME value fuseim.pri/ifs but it is good to change with your enviroment. for example myproject/nfs
	— name: PROVISIONER_NAME
	value: fuseim.pri/ifs
	— name: NFS_SERVER
	value: 172.16.2.5
	— name: NFS_PATH
	value: /ifs/storage
	volumes:
	— name: nfs-client-root
	nfs:
	server: 172.16.2.5
	path: /ifs/storage

# Edit deploy/class.yaml file
	set provisioner: fuseim.pri/ifs value as you defined PROVISIONER_NAME in deploy/deployment.yaml (myproject/nfs). It must match PROVISIONER_NAME otherwise it will fail!

# class.yaml defines the NFS-Client’s Kubernetes Storage Class with name: managed-nfs-storage

# We will define this storage-class name in claim yaml files later.

#
# Deploy and create storage class
#

	oc create -f deploy/class.yaml
	oc create -f deploy/deployment.yaml

# Check nfs-client-provisioner pod. Ensure that nfs-client-provisioner is running.
	oc get pods
	NAME                                    READY STATUS RESTARTS AGE
	nfs-client-provisioner-6f59b7b5f4-sd7r2    1/1 Running 2      1h

# Check the logs if there is a failure. If you don’t add hostmount-anyuid above, it will never work! Please double check it if you see an issue.
	oc logs nfs-client-provisioner-6f59b7b5f4-sd7r2

# Now we can create claim and test pod. You don’t have to mount NFS share on openshift master or nodes. nfs-client-provisioner will do it automatically and on demand.
