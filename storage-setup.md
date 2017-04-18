```sh

gcloud compute instances create "g1" --zone "asia-east1-a" --machine-type g1-small \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "g1"  \
  --private-network-ip 10.240.0.90  --subnet openshift-subnet

gcloud compute instances create "g2" --zone "asia-east1-a" --machine-type g1-small \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "g2"  \
  --private-network-ip 10.240.0.91  --subnet openshift-subnet
  
gcloud compute instances create "g3" --zone "asia-east1-a" --machine-type g1-small \
  --image "centos-7-v20170223" --image-project "centos-cloud" --boot-disk-size "20" \
  --boot-disk-type "pd-ssd" --boot-disk-device-name "g3"  \
  --private-network-ip 10.240.0.92  --subnet openshift-subnet
  
gcloud compute disks create g1-disk --zone asia-east1-a --type pd-ssd  --size 50GB
gcloud compute instances attach-disk g1-disk --disk  g1-disk  
  
gcloud compute disks create g2-disk --zone asia-east1-a --type pd-ssd  --size 50GB
gcloud compute instances attach-disk g2-disk --disk  g2-disk 

gcloud compute disks create g3-disk --zone asia-east1-a --type pd-ssd  --size 50GB
gcloud compute instances attach-disk g3-disk --disk  g3-disk 

##   Cleanup
gcloud compute instances detach-disk g1 --disk  g1-disk
gcloud compute instances detach-disk g2 --disk  g2-disk
gcloud compute instances detach-disk g3 --disk  g3-disk
gcloud compute  disks delete g1-disk g2-disk g3-disk

### on all g1,g2,g3
sudo yum install -y centos-release-gluster310.noarch
sudo yum install -y glusterfs gluster-cli glusterfs-libs glusterfs-server
sudo systemctl enable glusterd --now

#on g1
sudo yum install heketi heketi-client 
