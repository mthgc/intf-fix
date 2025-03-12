# intf-fix
Arquivos para atualização e auxílio na instalação do P4-INT (In-band network telemetry) usando ONOS and eBPF do  
## Sumário

1. Instalar todas as dependências
2. Instalar todos os pacotes necessários
3. Executar ONOS
4. Executar Mininet com BMv2
5. Executar INT Apps
6. Executar Parser, XDP & eBPF
7. Adicionar Dados ao Grafana
8. Verificação

## Requisitos do Sistema

Certifique-se de que o kernel e a versão do Ubuntu sejam compatíveis. Versões superiores podem funcionar, mas a versão abaixo já foi testada.

## Preparação

### 1. Clonar Repositório
```shell
sudo su
cd
apt update
git clone https://github.com/opennetworkinglab/onos.git -b onos-2.1
```


### 2. Executar Root-bootstrap

Essa atividade instalará automaticamente:
- Dependências
- Criação do usuário SDN
- Instalação do Bazel

Baixe o script root-bootstrap.sh:
```bash
cd /root/onos/tools/dev/p4vm
mv root-bootstrap.sh root-bootstrap.sh.bak 
sudo nano root-bootstrap.sh
```
Copie e cole o conteúdo do [Github](https://github.com/assyafii/ONOS-P4-INT/blob/main/root-bootstrap.sh)

Execute o script:
```bash
bash root-bootstrap.sh
```

### 3. Executar User-bootstrap


```shell
su sdn
#pwd: rocks
sudo apt-get install -y --no-install-recommends autoconf automake bison build-essential cmake cpp curl flex git graphviz libavl-dev libboost-dev libboost-graph-dev libboost-program-options-dev libboost-system-dev libboost-filesystem-dev libboost-thread-dev libc6-dev libev-dev libevent-dev libffi-dev libfl-dev libgc-dev libgc1c2 libgflags-dev libgmp-dev libgmp10 libgmpxx4ldbl libjudy-dev libpcap-dev libpcre3-dev libssl-dev libtool make pkg-config python2.7 python2.7-dev tcpdump wget unzip
sudo -H pip2.7 install setuptools cffi ipaddr ipaddress pypcap ptf git+https://github.com/p4lang/scapy-vxlan 
pip2.7 install coverage==5.5 cython==0.29.36 enum34==1.1.10 futures==3.4.0

```

Baixe o script user-bootstrap.sh:
```bash
cd /root/onos/tools/dev/p4vm
mv user-bootstrap.sh user-bootstrap.sh.bak
sudo nano user-bootstrap.sh
```
Copie e cole o conteúdo do [Github](https://github.com/assyafii/ONOS-P4-INT/blob/main/user-bootstrap.sh)

Execute o script:
```bash
su sdn 'user-bootstrap.sh'
```

### 4. Compilar ONOS
```bash
su sdn
cd ~/onos/
sed -i 's/http:/https:/g' ~/onos/tools/build/bazel/generate_workspace.bzl
sudo cat ~/onos/tools/build/bazel/generate_workspace.bzl | grep https:
bazel run onos-local clean
```

Acesse o painel ONOS:
```
http://ip_onos:8181/onos/ui/index.html
Usuário: onos
Senha: rocks
```

### 5. Compilar BCC (BPF Compiler Collection)
```bash
sudo apt -y install bison build-essential cmake flex git libedit-dev libllvm3.9 llvm-3.9-dev libclang-3.9-dev python zlib1g-dev libelf-dev clang-3.9
wget https://github.com/iovisor/bcc/archive/refs/heads/tag_v0.7.0.zip
unzip tag_v0.7.0.zip
sudo mv bcc-tag_v0.7.0 bcc 
mkdir bcc/build; cd bcc/build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr
make
sudo make install
```

### 6. Baixar BPF Collector
```bash
cd ~
git clone https://gitlab.com/tunv_ebpf/BPFCollector.git
cd BPFCollector
git checkout -t origin/spec_1.0
```

### 7. Ativar BPF e Criar Interface Virtual
```bash
sudo sysctl net/core/bpf_jit_enable=1
pip install cython
sudo ip link add veth_1 type veth peer name veth_2 
sudo ip link set dev veth_1 up 
sudo ip link set dev veth_2 up 
```

## Seção de Monitoramento

### 1. Instalar InfluxDB
```bash
sudo curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
sudo echo "deb https://repos.influxdata.com/ubuntu bionic stable" | sudo tee /etc/apt/sources.list.d/influxdb.list
sudo apt update
sudo apt install influxdb
sudo systemctl enable --now influxdb
sudo systemctl status influxdb
```

### 2. Instalar Prometheus
```bash
pip install prometheus-client
wget https://s3-eu-west-1.amazonaws.com/deb.robustperception.io/41EFC99D.gpg | sudo apt-key add -
sudo apt update
sudo apt -y install prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

### 3. Adicionar Exportador INT ao Prometheus
```bash
sudo nano /etc/prometheus/prometheus.yml
```
Adicione:
```yaml
  - job_name: p4switch
    static_configs:
      - targets: ['localhost:8000']
```
Reinicie o serviço:
```bash
sudo systemctl restart prometheus
```

### 4. Instalar Grafana
```bash
sudo apt-get install gnupg2 curl software-properties-common -y
curl https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update -y
sudo apt-get install grafana -y
sudo systemctl enable --now grafana-server
```
Acesse Grafana:
```
http://your_ip:3000
Usuário: admin
Senha: admin
```

## Execução do ONOS-INT P4

Essa seção requer 4 terminais.

### Terminal 1 (Executar ONOS)
```bash
su sdn
cd ~/onos/
bazel run onos-local clean
```
Ative as aplicações ONOS via Dashboard:
- Reactive Forwarding
- Proxy ARP
- INT Dashboard

### Terminal 2 (Executar Mininet-BMv2)
```bash
su sdn
cd ~
sudo -E $ONOS_ROOT/tools/test/topos/bmv2-demo.py --onos-ip=127.0.0.1 --pipeconf-id=org.onosproject.pipelines.int
```

### Terminal 3 (Executar BPF Parser)
```bash
su sdn 
cd ~ 
sudo python BPFCollector/PTClient.py veth_2
```

### Terminal 4 (Adicionar Mirroring)
```bash
su sdn 
cd ~ 
simple_switch_CLI --thrift-port `cat /tmp/bmv2-s12-thrift-port`
mirroring_add 500 4
```

## Geração de Tráfego
```bash
h2 iperf -s -u &
h1 iperf -c h2 -u -t 1000 -i 1 
```

## Verificação da Rede e Adição de Fluxo INT
```bash
cd ~/BPFCollector
sudo python -m pytest 
```

## Visualização dos Métricos no Grafana
1. Adicione um novo painel
2. Escolha a métrica desejada
