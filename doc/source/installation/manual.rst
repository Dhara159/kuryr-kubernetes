Installing kuryr-kubernetes manually
====================================

Configure kuryr-k8s-controller
------------------------------

Install ``kuryr-k8s-controller`` in a virtualenv::

    $ mkdir kuryr-k8s-controller
    $ cd kuryr-k8s-controller
    $ virtualenv env
    $ git clone http://git.openstack.org/openstack/kuryr-kubernetes
    $ . env/bin/activate
    $ pip install -e kuryr-kubernetes


In neutron or in horizon create subnet for pods, subnet for services and a
security-group for pods. You may use existing if you like.

.. todo::
    Add reference neutron cli commands

Create ``/etc/kuryr/kuryr.conf``::

    $ cd kuryr-kubernetes
    $ ./tools/generate_config_file_samples.sh
    $ cp etc/kuryr.conf.sample /etc/kuryr/kuryr.conf

Edit ``kuryr.conf``::

    [DEFAULT]
    use_stderr = true
    bindir = {path_to_env}/libexec/kuryr

    [kubernetes]
    api_root = http://{ip_of_kubernetes_apiserver}:8080

    [neutron]
    auth_url = http://127.0.0.1:35357/v3/
    username = admin
    user_domain_name = Default
    password = ADMIN_PASSWORD
    project_name = service
    project_domain_name = Default
    auth_type = password

    [neutron_defaults]
    ovs_bridge = br-int
    pod_security_groups = {id_of_secuirity_group_for_pods}
    pod_subnet = {id_of_subnet_for_pods}
    project = {id_of_project}
    service_subnet = {id_of_subnet_for_k8s_services}

Note that the service_subnet and the pod_subnet *should be routable* and that
the pods should allow service subnet access.

Octavia supports two ways of performing the load balancing between the
Kubernetes load balancers and their members:

* Layer2: Octavia, apart from the VIP port in the services subnet, creates a
  Neutron port to the subnet of each of the members.  This way the traffic from
  the Service Haproxy to the members will not go through the router again, only
  will have gone through the router to reach the service.
* Layer3: Octavia only creates the VIP port. The traffic from the service VIP to
  the members will go back to the router to reach the pod subnet. It is
  important to note that it will have some performance impact depending on the SDN.

At the moment Kuryr-Kubernetes supports only L3 mode (both for Octavia and for
the deprecated Neutron-LBaaSv2.

This means that:

* There should be a router between the two subnets.
* The pod_security_groups setting should include a security group with a rule
  granting access to all the CIDR or the service subnet, e.g.::

    openstack security group create --project k8s_cluster_project \
        service_pod_access_sg
    openstack --project k8s_cluster_project security group rule create \
        --remote-ip cidr_of_service_subnet --ethertype IPv4 --protocol tcp \
        service_pod_access_sg

* The uuid of this security group id should be added to the comma separated
  list of pod security groups. *pod_security_groups* in *[neutron_defaults]*.

Run kuryr-k8s-controller::

    $ kuryr-k8s-controller --config-file /etc/kuryr/kuryr.conf -d

Alternatively you may run it in screen::

    $ screen -dm kuryr-k8s-controller --config-file /etc/kuryr/kuryr.conf -d

Configure kuryr-cni
-------------------

On every kubernetes minion node (and on master if you intend to run containers
there) you need to configure kuryr-cni.

Install ``kuryr-cni`` a virtualenv::

    $ mkdir kuryr-k8s-cni
    $ cd kuryr-k8s-cni
    $ virtualenv env
    $ . env/bin/activate
    $ git clone http://git.openstack.org/openstack/kuryr-kubernetes
    $ pip install -e kuryr-kubernetes

Create ``/etc/kuryr/kuryr.conf``::

    $ cd kuryr-kubernetes
    $ ./tools/generate_config_file_samples.sh
    $ cp etc/kuryr.conf.sample /etc/kuryr/kuryr.conf

Edit ``kuryr.conf``::

    [DEFAULT]
    use_stderr = true
    bindir = /path/to/env/libexec/kuryr
    [kubernetes]
    api_root = http://{ip_of_kubernetes_apiserver}:8080

Link the CNI binary to CNI directory, where kubelet would find it::

    $ mkdir -p /opt/cni/bin
    $ ln -s $(which kuryr-cni) /opt/cni/bin/

Create the CNI config file for kuryr-cni: ``/etc/cni/net.d/10-kuryr.conf``.
Kubelet would only use the lexicographically first file in that directory, so
make sure that it is kuryr's config file::

    {
        "cniVersion": "0.3.0",
        "name": "kuryr",
        "type": "kuryr-cni",
        "kuryr_conf": "/etc/kuryr/kuryr.conf",
        "debug": true
    }

Install ``os-vif`` and ``oslo.privsep`` libraries globally. These modules
are used to plug interfaces and would be run with raised privileges. ``os-vif``
uses ``sudo`` to raise privileges, and they would need to be installed globally
to work correctly::

    deactivate
    sudo pip install 'oslo.privsep>=1.20.0' 'os-vif>=1.5.0'
