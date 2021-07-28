# containerized-nginx

This repository is about containerizing nginx and adding a second interface to the pod and exposing applications without creating any service in OCP. This solution was tested on VMWare 6.7 and OCP 4.7.19

# Prequisites

1. Running OCP 4.7.x cluster
2. NMState operator installed
3. VLAN defined in your network

# Adding second interface to worker nodes

1. Create a trunk port in esxi. Make sure that VLAN ID is set to 4095, also enable promiscuous mode

   ![Trunk Port](https://github.com/rh-telco-tigers/containerized-nginx/raw/main/images/trunk-port.png)

2. Add second interface to worker node using the port created above

   ![Trunk Port](https://github.com/rh-telco-tigers/containerized-nginx/raw/main/images/second-nic.png)

3. Configure second nic on worker node using nmstate operator
```yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: second-interface-noip
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ''
  desiredState:
    interfaces:
    - name: ens224
      description: no ip on ens224
      type: ethernet
      state: up
      ipv4:
        enabled: false
```

4. Create a vlan tagged interface for example vlan 21
```yaml
apiVersion: nmstate.io/v1alpha1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vlan21-ens224
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ''
  desiredState:
    interfaces:
    - name: ens224.21
      description: VLAN using ens224
      type: vlan
      state: up
      vlan:
        base-iface: ens224
        id: 21
```

5. Create a project
```bash
#oc new-project haproxy-demo
```

6. Create a network attached definition in the namespace haproxy-demo
```yaml
#oc edit networks.operator.openshift.io cluster
spec:
  additionalNetworks:
  - name: demo-21
    namespace: haproxy-demo
    simpleMacvlanConfig:
      ipamConfig:
        type: dhcp
      master: ens224.21
      mode: bridge
    type: SimpleMacvlan
  clusterNetwork:
....
```

7. Create two pods for application app1 app2
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1
  namespace: haproxy-demo
spec:
  containers:
  - name: app1
    image: quay.io/ashisha/rogers-app1
---
apiVersion: v1
kind: Pod
metadata:
  name: app2
  namespace: haproxy-demo
spec:
  containers:
  - name: app2
    image: quay.io/ashisha/rogers-app2
```
Note the ip of the pods created above
```
oc get pod app1 -o json | jq .status.podIP
"10.129.2.205"
oc get pod app2 -o json | jq .status.podIP
"10.129.1.147"
```

8. Create a default nginx configuration name the file as default.conf
```
upstream loadbalancer {
server 10.131.1.147:5000 weight=5;
server 10.129.2.205:5000 weight=5;
}
server {
location / {
proxy_pass http://loadbalancer;
}}
```

9. Create a configmap to store the above nginx confiuration
```
#oc create configmap nginx-conf --from-file default.conf
```

10. Create the nginx pod with multus
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-lb
  namespace: haproxy-demo
  annotations:
    k8s.v1.cni.cncf.io/networks: |-
      [
        {
          "name": "demo-21",
          "namespace": "haproxy-demo",
          "default-route": ["192.168.21.1"]
        }
      ]
spec:
  containers:
  - name: nginx-lb
    image: quay.io/ashisha/nginx-lb
    volumeMounts:
    - name: "config-volume"
      mountPath: "/etc/nginx/conf.d/default.conf"
      subPath: "default.conf"
  volumes:
  - name: "config-volume"
    configMap:
      name: "nginx-conf"
    restartPolicy: Never
```

11. make sure all the pod are running
```
#oc get pods
NAME       READY   STATUS    RESTARTS   AGE
app1       1/1     Running   0          16h
app2       1/1     Running   0          16h
nginx-lb   1/1     Running   0          4h53m
```

12. Get the nginx pod ip
```
#oc describe po nginx-lb
Name:         nginx-lb
Namespace:    haproxy-demo
Priority:     0
Node:         hulk-gjvr9-worker-59wfh/192.168.20.156
Start Time:   Thu, 22 Jul 2021 12:01:48 -0400
Labels:       <none>
Annotations:  k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "",
                    "interface": "eth0",
                    "ips": [
                        "10.129.3.179"
                    ],
                    "default": true,
                    "dns": {}
                },{
                    "name": "haproxy-demo/demo-21",
                    "interface": "net1",
                    "ips": [
                        "192.168.21.22"
                    ],
                    "mac": "3a:b1:0a:d8:7b:28",
                    "dns": {}
                }]
...
```

13. Curl the ip to see if LB is working
```
#curl http://192.168.21.22
Hello World, this is App2 :)
#curl http://192.168.21.22
Hello World, this is App1 :)
```




