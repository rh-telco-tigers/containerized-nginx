# containerized-nginx

This repository is about containerizing nginx and adding a second interface to the pod and exposing applications without creating any service in OCP. This solution was tested on VMWare 6.7 and OCP 4.7.19

# Adding second interface to worker nodes

1. Create a trunk port in esxi
   ![Trunk Port](https://github.com/rh-telco-tigers/containerized-nginx/raw/main/images/trunk-port.png)
