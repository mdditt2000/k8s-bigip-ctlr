#access the IMI Shell
imish

#Switch to enable mode
enable

#Enter configuration mode
config terminal

#Setup route bgp with AS Number 65000
router bgp 65000

#Create BGP Peer group
neighbor kube-vip-k8s peer-group

#assign peer group as BGP neighbors
neighbor kube-vip-k8s remote-as 65000

#we need to add all the peers: the other BIG-IP, our k8s components
# For Ipv6, provide relevant ip address
neighbor 192.168.200.61 peer-group kube-vip-k8s

#save configuration
write

#exit
end