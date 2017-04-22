# cloud.xfc.io
```sh
gcloud compute instances list 

gcloud config set compute/region asia-east1

gcloud config set compute/zone asia-east1-a

gcloud compute networks create openshift --mode custom

gcloud compute networks subnets create openshift-subnet \
  --network openshift \
  --range 10.240.0.0/24
  

gcloud compute firewall-rules create allow-internal \
  --allow tcp,udp,icmp \
  --network openshift \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
  
 
gcloud compute firewall-rules create allow-external \
  --allow tcp:22,tcp:3389,tcp:6443,tcp:8443,icmp \
  --network openshift \
  --source-ranges 0.0.0.0/0  
  
  
gcloud compute firewall-rules create allow-healthz \
  --allow tcp:8080 \
  --network openshift \
  --source-ranges 130.211.0.0/22 
  

gcloud compute firewall-rules create allow-http-https   --allow tcp:8080,tcp:8443,tcp:443,tcp:80   --network openshift   --source-ranges 0.0.0.0/0

gcloud compute firewall-rules list --filter "network=openshift"

gcloud compute addresses create openshift --region=asia-east1

gcloud compute addresses list openshift

gcloud compute instances create "master1" --zone "asia-east1-a" --machine-type n1-standard-1 \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "master1"  \
  --private-network-ip 10.240.0.10  --subnet openshift-subnet
  

gcloud compute instances create "master2" --zone "asia-east1-a" --machine-type n1-standard-1 \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "master2"  \
  --private-network-ip 10.240.0.11  --subnet openshift-subnet
  
  
gcloud compute instances create "master3" --zone "asia-east1-a" --machine-type n1-standard-1 \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "master3"  \
  --private-network-ip 10.240.0.12  --subnet openshift-subnet  

 gcloud compute instances create "infra1" --zone "asia-east1-a" --machine-type n1-standard-1  \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "infra1"  \
  --private-network-ip 10.240.0.51  --subnet openshift-subnet   
  

 gcloud compute instances create "infra2" --zone "asia-east1-a" --machine-type n1-standard-1  \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "infra2"  \
  --private-network-ip 10.240.0.52  --subnet openshift-subnet  
  
  
gcloud compute instances create "node1" --zone "asia-east1-a" --machine-type n1-standard-1  \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "node1"  \
  --private-network-ip 10.240.0.75  --subnet openshift-subnet  


gcloud compute addresses create openshift-ip --region=asia-east1


gcloud compute http-health-checks create openshift-api-health-check \
  --description "Openshift API Server Health Check" \
  --port 8443 \
  --request-path /healthz


gcloud compute target-pools create openshift-api-target-pool \
  --http-health-check=openshift-api-health-check


gcloud compute target-pools add-instances openshift-api-target-pool \
  --instances master1,master2,master3   


OPENSHIFT_PUBLIC_ADDRESS=$(gcloud compute addresses describe openshift-ip \
  --region asia-east1 \
  --format 'value(address)') 

gcloud compute forwarding-rules create kubernetes-forwarding-rule \
  --address ${OPENSHIFT_PUBLIC_ADDRESS} \
  --ports 8443 \
  --target-pool openshift-api-target-pool \
  --region asia-east1
  
echo $OPENSHIFT_PUBLIC_ADDRESS > OPENSHIFT_PUBLIC_ADDRESS

gcloud compute copy-files ~/.ssh/id_rsa master1:~/
gcloud compute copy-files OPENSHIFT_PUBLIC_ADDRESS master1:~/

gcloud compute ssh master1
ssh-agent $SHELL
ssh-add ~/id_rsa
export OPENSHIFT_PUBLIC_ADDRESS=$(cat OPENSHIFT_PUBLIC_ADDRESS)

sudo yum install -y wget git ansible pyOpenSSL python-cryptography python-lxml

wget https://raw.githubusercontent.com/xfcio/cloud.xfc.io/master/inventory.yaml 

sed -e s/OPENSHIFT_IP/${OPENSHIFT_PUBLIC_ADDRESS}/g inventory.yaml > inventory.current.yaml


git clone https://github.com/openshift/openshift-ansible
cd openshift-ansible/
git checkout release-1.5
cd ~
export ANSIBLE_HOST_KEY_CHECKING=False
ansible-playbook -i inventory.current.yaml openshift-ansible/playbooks/byo/config.yml
sudo yum install -y heketi-client
#pod join network? 
ansible all -m shell -a "sudo iptables -I INPUT -p all -j ACCEPT" -i inventory.current.yaml
./gk-deploy -g -t ./ocp-templates -n cns -c oc --admin-key Gtfer452 --user-key g6r6na --load -v topology.json
```

`oc create -f storageclass.yaml -n deffault` 

```yaml
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: default
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://heketi-cns.router.default.svc.cluster.local"
  restuser: "admin"
  restuserkey: "adminpass"
```


>  Extras 1
```sh
ansible all -m shell -a "sudo cp /etc/origin/master/master-config.yaml /etc/origin/master/master-config.yaml.bak" -i cloud.xfc.io/inventory.yaml
ansible all -m shell -a "sudo sed -i s/35.187.153.182/cloud.xfc.io/g /etc/origin/master/master-config.yaml" -i cloud.xfc.io/inventory.yaml
ansible all -m shell -a "sudo sed -i s/oldsecret/newsecret/g /etc/origin/master/master-config.yaml" -i cloud.xfc.io/inventory.yaml
ansible all -m shell -a "sudo systemctl restart origin-master-api" -i cloud.xfc.io/inventory.yaml
ansible all -m shell -a "sudo systemctl restart origin-master-controllers" -i cloud.xfc.io/inventory.yaml

##Caution
ansible all -m shell -a "sudo wipefs -a /dev/sdb" -i ~/cloud.xfc.io/inventory.yaml

./gk-deploy -g -t ./ocp-templates -n cns -c oc --admin-key adminpass --user-key userpass --load -v topology.json
```


### Extras
```sh
gcloud compute disks create master-1-disk --zone asia-east1-a --type pd-ssd  --size 50GB
gcloud compute instances attach-disk master1 --disk  master-1-disk

gcloud compute disks create master-2-disk --zone asia-east1-a --type pd-ssd  --size 50GB
gcloud compute instances attach-disk master2 --disk  master-2-disk

gcloud compute disks create master-3-disk --zone asia-east1-a --type pd-ssd  --size 50GB
gcloud compute instances attach-disk master3 --disk  master-3-disk


gcloud compute instances detach-disk master1 --disk  master-1-disk
gcloud compute instances detach-disk master2 --disk  master-2-disk
gcloud compute instances detach-disk master3 --disk  master-3-disk

gcloud compute instances detach-disk node1 --disk  node-1-disk
gcloud compute instances detach-disk infra1 --disk  infra-1-disk
gcloud compute instances detach-disk infra2 --disk  infra-2-disk

gcloud compute  disks delete master-1-disk master-2-disk  master-3-disk node-1-disk infra-1-disk infra-2-disk

```

### Cleanup 
```
gcloud compute firewall-rules delete $(gcloud compute firewall-rules list --filter "network=openshift" | awk '{print $1}')
gcloud compute forwarding-rules delete xfc-masters-ig --region asia-east1
gcloud compute networks delete openshift

```
