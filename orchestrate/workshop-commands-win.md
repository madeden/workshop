##### Installing the Prerequisites #####
# AWS CLI
http://docs.aws.amazon.com/cli/latest/userguide/installing.html

# JQ
https://stedolan.github.io/jq/download/

# kubectl
https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/windows/amd64/kubectl.exe

# Helm 
https://kubernetes-helm.storage.googleapis.com/helm-canary-windows-amd64.zip


##### Installing Juju #####

https://launchpad.net/juju/2.1/2.1.2/+download/juju-setup-2.1.2.exe

https://jujucharms.com/docs/stable/getting-started-keygen-win

##### display help #####
juju help 
juju help commands


##### Adding Credentials #####
# Listing & Updating Clouds
juju clouds
juju update-clouds


# Adding credentials for AWS
juju add-credential aws


##### Bootstrapping #####
juju bootstrap aws/eu-central-1


##### Adding a model #####
juju add-model k8s
juju model-config resource-tags="KubernetesCluster=workshop"


##### Deploying a Development version of k8s #####
juju deploy cs:bundle/kubernetes-core-15


##### Deployment tracking commands #####
# Status
juju status --color

# Track Status
watch -c juju status --color

# Tail logs
juju debug-log

# Access the GUI
juju gui --show-credentials

##### Gathering configuration #####
mkdir -p .kube
juju scp kubernetes-master/0:config ~/.kube/config

kubectl.exe cluster-info # Default Credentials: admin/admin
# Kubernetes master is running at https://54.83.170.207:6443
# Heapster is running at https://54.83.170.207:6443/api/v1/proxy/namespaces/kube-system/services/heapster
# KubeDNS is running at https://54.83.170.207:6443/api/v1/proxy/namespaces/kube-system/services/kube-dns
# kubernetes-dashboard is running at https://54.83.170.207:6443/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
# Grafana is running at https://54.83.170.207:6443/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana
# InfluxDB is running at https://54.83.170.207:6443/api/v1/proxy/namespaces/kube-system/services/monitoring-influxdb


##### Useful Juju Commands #####
# Logging to a machine
juju ssh kubernetes-master/0 
juju ssh 0


# Executing a command on remote machine
juju ssh kubernetes-master/0 "sudo cat /proc/cpuinfo"
# processor : 0
# vendor_id : GenuineIntel
# cpu family  : 6
# model   : 62
# model name  : Intel(R) Xeon(R) CPU E5-2670 v2 @ 2.50GHz
# stepping  : 4
# microcode : 0x416
# cpu MHz   : 2500.082
# cache size  : 25600 KB
# physical id : 0
# siblings  : 1
# core id   : 0
# cpu cores : 1
# apicid    : 0
# initial apicid  : 0
# fpu   : yes
# fpu_exception : yes
# cpuid level : 13
# wp    : yes
# flags   : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl xtopology eagerfpu pni pclmulqdq ssse3 cx16 pcid sse4_1 sse4_2 x2apic popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm fsgsbase smep erms xsaveopt
# bugs    :
# bogomips  : 5000.16
# clflush size  : 64
# cache_alignment : 64
# address sizes : 46 bits physical, 48 bits virtual
# power management:

# Downloading a file
juju scp kubernetes-master/0:config ./


# Uploading a file
touch foo.txt
juju scp ./foo.txt kubernetes-master/0:test.txt

# List operational actions available 
juju actions kubernetes-worker
# Action            Description
# clean-containers  Garbage collect non-running containers
# clean-images      Garbage collect non-running images
# debug             Collect debug data
# microbot          Launch microbot containers
# pause             Cordon the unit, draining all active workloads.
# resume            UnCordon the unit, enabling workload scheduling.

# Executing an action
juju run-action kubernetes-worker/0 microbot replicas=5
# Action queued with id: 2184f1f5-4dea-4e34-8e1c-20778337f5d2

# Looking at the output
$ juju show-action-output 2184f1f5-4dea-4e34-8e1c-20778337f5d2
# results:
#   address: microbot.52.202.51.82.xip.io
# status: completed
# timing:
#   completed: 2017-04-05 07:29:35 +0000 UTC
#   enqueued: 2017-04-05 07:29:31 +0000 UTC
#   started: 2017-04-05 07:29:33 +0000 UTC


##### Add Support for ELBs #####

aws ec2 describe-instances \
  --filters "Name=tag:juju-units-deployed,Values=*kubernetes-master*" | \
  jq --raw-output '.[][].Instances[].InstanceId' | \
  xargs -I {} aws ec2 associate-iam-instance-profile \
    --iam-instance-profile Name=k8sMaster-Instance-Profile \
    --instance-id {}

aws ec2 describe-instances --filters "Name=tag:juju-units-deployed,Values=kubernetes-worker*" | \
  jq --raw-output '.[][].Instances[].InstanceId' | \
  xargs -I {} aws ec2 associate-iam-instance-profile --iam-instance-profile Name=k8sWorker-Instance-Profile --instance-id {}

for KEY in KubernetesCluster juju-controller-uuid juju-model-uuid; do
  aws ec2 describe-instances | \
    jq --raw-output '.[][].Instances[] | select( .Tags[].Value  | contains ("k8s")) | .SecurityGroups[1].GroupId' | \
    sort | uniq | \
    xargs -I {} aws ec2 delete-tags --resources {}  --tags "Key="${KEY}
done

aws ec2 describe-instances | \
  jq --raw-output '.[][].Instances[] | select( .Tags[].Value  | contains ("k8s")) | .SecurityGroups[0].GroupId' | \
  sort | uniq | \
  xargs -I {} aws ec2 delete-tags --resources {} --tags "Key=juju-model-uuid"

aws ec2 describe-instances | \
  jq --raw-output '.[][].Instances[] | select( .Tags[].Value  | contains ("k8s")) | .SecurityGroups[0].GroupId' | \
  sort | uniq | \
  xargs -I {} aws ec2 delete-tags --resources {} --tags "Key=juju-controller-uuid"

juju status kubernetes-worker --format json | \
  jq -r '.machines[]."instance-id"' | \
  xargs -I {} aws ec2 describe-instances --instance-ids {} | \
  jq --raw-output '.[][].Instances[].SubnetId' | sort | uniq | \
  xargs -I {} aws ec2 create-tags --resources {} --tags "Key=KubernetesCluster,Value=workshop"

juju show-status kubernetes-master --format json | \
  jq -r '.applications."kubernetes-master".units | keys[]' | \
  xargs -I UNIT juju ssh UNIT "sudo sed -i 's/KUBE_CONTROLLER_MANAGER_ARGS=\"/KUBE_CONTROLLER_MANAGER_ARGS=\"--cloud-provider=aws\ /' /etc/default/kube-controller-manager && sudo systemctl restart kube-controller-manager.service"

juju show-status kubernetes-master --format json | \
  jq -r '.applications."kubernetes-master".units | keys[]' | \
  xargs -I UNIT juju ssh UNIT "sudo sed -i 's/KUBE_API_ARGS=\"/KUBE_API_ARGS=\"--cloud-provider=aws\ /' /etc/default/kube-apiserver && sudo systemctl restart kube-apiserver.service"

juju show-status kubernetes-worker --format json | \
  jq -r '.applications."kubernetes-worker".units | keys[]' | \
  xargs -I UNIT juju ssh UNIT "sudo sed -i 's/KUBELET_ARGS=\"/KUBELET_ARGS=\"--cloud-provider=aws\ /' /etc/default/kubelet && sudo systemctl restart kubelet.service"

kubectl.exe run hello-world \
--replicas=5 \
--labels="run=load-balancer-example" \
--image=gcr.io/google-samples/node-hello:1.0 \
--port=8080

kubectl.exe expose deployment hello-world --type=LoadBalancer --name=hello

##### Helm #####
# Initialize Helm
helm.exe init
helm.exe repo update

# Install Wordpress with ELB
helm.exe install --name wordpress --set wordpressUsername=admin,wordpressPassword=password,mariadb.mariadbRootPassword=secretpassword,persistence.enabled=false kubernetes-charts/wordpress

kubectl.exe get pods

kubectl.exe get secret --namespace default wordpress-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode

kubectl.exe get svc -o wide

# Install Wordpress with Ingress
helm.exe install --name wordpress-ingress --set wordpressUsername=admin,wordpressPassword=password,mariadb.mariadbRootPassword=secretpassword,persistence.enabled=false,serviceType=ClusterIP,ingress.enabled=true,ingress.hostname=www.34.208.240.229.xip.io kubernetes-charts/wordpress

kubectl.exe get ing -o wide

# Scale Application
kubectl.exe get deploy
# NAME                          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
# hello-world                   5         5         5            5           1h
# wordpress-ingress-mariadb     1         1         1            1           17m
# wordpress-ingress-wordpress   1         1         1            1           17m

kubectl.exe scale deploy wordpress-ingress-wordpress --replicas=3
# deployment "wordpress-ingress-wordpress" scaled

kubectl.exe get pods -o wide
# NAME                                           READY     STATUS    RESTARTS   AGE       IP              NODE
# wordpress-ingress-mariadb-4241180659-drs87     1/1       Running             0          21m       10.1.61.9       ip-172-31-19-23.us-west-2.compute.internal
# wordpress-ingress-wordpress-2693923398-5pr10   1/1       Running             0          15m       10.1.61.10      ip-172-31-19-23.us-west-2.compute.internal
# wordpress-ingress-wordpress-2693923398-pq329   0/1       Running             0          1s        10.1.58.4       ip-172-31-7-167.us-west-2.compute.internal
# wordpress-ingress-wordpress-2693923398-qzwtk   1/1       Running             0          3m        10.1.6.16       ip-172-31-43-243.us-west-2.compute.internal

##### Get Workshop Name #####
juju status --format json | jq -r '.machines[]."instance-id"' | head -n1 | xargs -I {} aws ec2 describe-instances --instance-ids {} | jq '.[][].Instances[].Tags'
# {
#   "Value": "cacf0212-383d-44df-8f46-5a090239e1fa",
#   "Key": "juju-model-uuid"
# }
# {
#   "Value": "workshop",
#   "Key": "KubernetesCluster"
# }
# {
#   "Value": "juju-k8s-machine-0",
#   "Key": "Name"
# }
# {
#   "Value": "adcc2879-a1e8-4abf-8e76-c356b34620ac",
#   "Key": "juju-controller-uuid"
# }
# {
#   "Value": "etcd/0 kubernetes-master/0",
#   "Key": "juju-units-deployed"
# }


##### Adding an EFS Drive #####
# Filter: --filters Name=tag:KubernetesCluster,Value=workshop

VPC_ID=$(aws ec2 describe-vpcs | jq -r '.[][].VpcId')
echo $VPC_ID

SUBNET_IDS=$(aws ec2 describe-subnets | jq -r '.[][].SubnetId')
echo $SUBNET_IDS

SG_ID="$(aws ec2 describe-instances \
  | jq -r '.[][].Instances[] | select( .Tags[].Value  | contains ("'"k8s"'")) | .SecurityGroups[1].GroupId' | sort | uniq | tr '\n' ' ')"
echo $SG_ID
# Note: k8s here is the model name

EFS_ID=$(aws efs create-file-system --creation-token $(uuid) jq -r '.FileSystemId')
echo $EFS_ID

for SUBNET in ${SUBNET_IDS}
do 
  aws efs create-mount-target \
    --file-system-id ${EFS_ID} \
    --subnet-id ${SUBNET} \
    --security-groups ${SG_ID}
done


