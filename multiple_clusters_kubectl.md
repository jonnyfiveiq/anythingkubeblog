# anythingkube
Connecting to multiple clusters with Kubectl

I'm studying for my CKA (Certified Kubernetes Administrator) and thought it prudent to learn how to make Kubectl connect to multiple clusters. This is documented here at [kubernetes.io](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) but I found it missing the dealing of certificates, especially when you just wish to use the default account at install. This article is going to show you how to take two Kubernetes clusters installed from Kubeadm and use just one Kubectl to administer both of them.

I have installed two Kubernetes clusters from Kubeadm and I shall name them as follows:

- **Cluster 1** - Production
- **Cluster 2** - Development

We shall use the Kubectl installed on the Production cluster to connect to the Development cluster. So we need to find out the details of the Development cluster.

Execute the following commands on the Development cluster.

This will return you the Cluster HTTP-Address and Port

    Development:$ kubectl cluster-info

    Kubernetes master is running at https://172.31.0.104:6443
    KubeDNS is running at https://172.31.0.104:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

Write down what you have as `https://172.31.0.104:6443`

Next we need to get the certificate files ca.crt, client.pem & client-key.pem

**client.pem**

    Development:$ export client=$(grep client-cert ~/.kube/config |cut -d" " -f 6)
    Development:$ echo $client | base64 -d - > ./client.pem

**client-key.pem**

    Development:$ export key=$(grep client-key-data ~/.kube/config |cut -d " " -f 6)
    Development:$ echo $key | base64 -d - > ./client-key.pem

**ca.crt**

    Development:$ export auth=$(grep certificate-authority-data ~/.kube/config |cut -d " " -f 6)
    Development:$ echo $auth | base64 -d - > ./ca.pem

You will now have the 3 files ca.crt, client.pem & client-key.pem in your current directory, decoded from base64. You need to transfer these to your server where you wish to enable Kubectl for mulitple cluster access, in my case this is my master server in the Production cluster.

We need to run 3 Kubectl commands to update the `~/.kube/config` file with a new cluster registration, user certificates and a context. Here are the commands that we need to run,

    Production:$ kubectl config set-cluster <NAME OF CLUSTER> --server=<CLUSTER IP:PORT> --certificate-authority=./ca.pem
    Production:$ kubectl config set-credentials <NAME OF USER> --client-certificate=./client.pem --client-key=./client-key.pem
    Production:$ kubectl config set-context <NAME OF CONTEXT> --cluster=<NAME OF CLUSTER> --namespace=default --user=<NAME OF USER>

Here are the same commands substituted with my details needed to make my context work.
I have called my cluster and context both `development`.  My user is named `jonny`

    Production:$ kubectl config set-cluster development --server=https://172.31.0.104:6443 --certificate-authority=./ca.crt
    Production:$ kubectl config set-credentials jonny --client-certificate=./client.pem --client-key=./client-key.pem
    Production:$ kubectl config set-context development --cluster=development --namespace=default --user=jonny

Now we can see what contexts are configured

    Production:$ kubectl config get-contexts

    CURRENT   NAME                          CLUSTER       AUTHINFO           NAMESPACE
    *         kubernetes-admin@kubernetes   kubernetes    kubernetes-admin   
              development                   development   jonny              default

You can see the current context is the local kubernetes cluster being Production, and also the new context we added called `development`

To switch and use the new `development` context run the following

    Production:$ kubectl config use-context development

Run the get-contexts command to see the CURRENT context switched

    CURRENT   NAME                          CLUSTER       AUTHINFO           NAMESPACE
              kubernetes-admin@kubernetes   kubernetes    kubernetes-admin   
    *         development                   development   jonny              default

Now you should be able to run any Kubectl command and it will be the remote cluster, in my case Development such as;

    Production:$ kubectl cluster-info

    Kubernetes master is running at https://172.31.0.104:6443
    KubeDNS is running at https://172.31.0.104:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

Hope this article helps you.
