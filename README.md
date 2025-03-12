# intf-fix
Files and documents to help you install an SDN scenario with BPF Collector + ONOS + Mininet.


sudo su
cd
apt update
git clone https://github.com/opennetworkinglab/onos.git -b onos-2.1


This activity will install automatically packages bellow :

Installing all depedencies
creating sdn user
Installing bazel
Make sure you already on root user, this command will finished until 20–30 minutes, let’s create coffee first:


cd /root/onos/tools/dev/p4vm
mv root-bootstrap.sh root-bootstrap.sh.bak 
sudo nano root-bootstrap.sh

Note : Copy & paste from Github

bash root-bootstrap.sh