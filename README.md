# ipsec-demo
IPSec IKEv2 Site-to-Site demo with Certificate Authority

## 1. create vms (aws)

https://cloud-images.ubuntu.com/locator/ec2/

```sh
export AMI_ID='ami-0e5657f6d3c3ea350' # eu-west-1 ubuntu 18.04 amd64
export AWS_REGION='eu-west-1'
```

```sh
docker-machine create \
  --driver amazonec2 \
  --amazonec2-instance-type t3.small \
  --amazonec2-region ${AWS_REGION} \
  --amazonec2-root-size 30 \
  --amazonec2-ami ${AMI_ID} \
  node-main

docker-machine create \
  --driver amazonec2 \
  --amazonec2-instance-type t3.small \
  --amazonec2-region ${AWS_REGION} \
  --amazonec2-root-size 30 \
  --amazonec2-ami ${AMI_ID} \
  node-peer-a

docker-machine create \
  --driver amazonec2 \
  --amazonec2-instance-type t3.small \
  --amazonec2-region ${AWS_REGION} \
  --amazonec2-root-size 30 \
  --amazonec2-ami ${AMI_ID} \
  node-peer-b

SG_ID=$(aws ec2 describe-security-groups \
  --group-names docker-machine \
  --query 'SecurityGroups[*].GroupId' \
  --output text \
  --no-cli-pager \
  --region ${AWS_REGION})

aws ec2 authorize-security-group-ingress \
  --group-id ${SG_ID} \
  --source-group ${SG_ID} \
  --protocol udp \
  --port 500 \
  --no-cli-pager \
  --region ${AWS_REGION}

aws ec2 authorize-security-group-ingress \
  --group-id ${SG_ID} \
  --source-group ${SG_ID} \
  --protocol udp \
  --port 4500 \
  --no-cli-pager \
  --region ${AWS_REGION}

aws ec2 authorize-security-group-ingress \
  --group-id ${SG_ID} \
  --source-group ${SG_ID} \
  --protocol icmp \
  --port -1 \
  --no-cli-pager \
  --region ${AWS_REGION}
```

## 2. ssh, install packages and setup environment (on all nodes)

```sh
docker-machine ssh node-main
docker-machine ssh node-peer-a
docker-machine ssh node-peer-b
```

```sh
sudo apt update

sudo apt install \
  iproute2 \
  strongswan \
  strongswan-pki \
  ipsec-tools

git clone https://github.com/jpfe-tid/ipsec-demo.git

cd ipsec-demo
```

## 3. generate ca (on main node)

```sh
ipsec pki \
  --gen \
  --type rsa \
  --size 4096 \
  --outform pem \
    > ca-key.pem

ipsec pki \
  --self \
  --ca \
  --lifetime 3650 \
  --in ca-key.pem \
  --type rsa \
  --dn "CN=ACME" \
  --outform pem \
    > ca-cert.pem

# copy ca-cert.pem to all peer nodes
```

## 4. generate private key and csr (on all peer nodes)

```sh
export DEFAULT_INTERFACE='ens5' # node network interface 
```

```sh
ipsec pki \
  --gen \
  --type rsa \
  --size 4096 \
  --outform pem \
    > ${HOSTNAME}-key.pem


IP=$(ifconfig ${DEFAULT_INTERFACE} | \
  grep 'inet ' | \
  cut -d' ' -f 10)

ipsec pki \
  --req \
  --in ${HOSTNAME}-key.pem \
  --dn "CN=${IP}" \
  --san "${IP}" \
  --outform pem \
    > ${HOSTNAME}-csr.pem

more ${HOSTNAME}-csr.pem | cat

# copy csr to main node for signing
```

## 5. sign csr (on main node)

```sh
ipsec pki \
  --issue \
  --in node-peer-a-csr.pem \
  --type pkcs10 \
  --lifetime 1825 \
  --cacert ca-cert.pem \
  --cakey ca-key.pem \
  --flag serverAuth \
  --flag ikeIntermediate \
  --outform pem \
    > node-peer-a-cert.pem

ipsec pki \
  --issue \
  --in node-peer-b-csr.pem \
  --type pkcs10 \
  --lifetime 1825 \
  --cacert ca-cert.pem \
  --cakey ca-key.pem \
  --flag serverAuth \
  --flag ikeIntermediate \
  --outform pem \
    > node-peer-b-cert.pem

more node-peer-*-cert.pem | cat

# copy certificate to their corresponding peer node
```

## 6. copy certs and keys (on all peer nodes)

```sh
cp ca-cert.pem /etc/ipsec.d/cacerts/ca-cert.pem

cp ${HOSTNAME}-cert.pem /etc/ipsec.d/certs/${HOSTNAME}-cert.pem

cp ${HOSTNAME}-key.pem /etc/ipsec.d/private/${HOSTNAME}-key.pem

ls -lah \
  /etc/ipsec.d/cacerts \
  /etc/ipsec.d/certs \
  /etc/ipsec.d/private
```

## 7. copy conf (on all peer nodes)

```sh
export LEFT_IP='172.31.1.101' # node-peer-a IP
export LEFT_HOSTNAME='node-peer-a'
export RIGHT_IP='172.31.1.102' # node-peer-b IP
export RIGHT_HOSTNAME='node-peer-b'
export HOSTNAME
```

```sh
cat ipsec.conf| envsubst > /etc/ipsec.conf

cat ipsec.secrets| envsubst > /etc/ipsec.secrets

ls -lah \
  /etc/ipsec.secrets \
  /etc/ipsec.conf
```

## 8. setup internal ip addresses (on each peer node)

```sh
ip a a 10.0.1.1 dev lo # on node-peer-a (internal ip)
ip a a 10.0.2.1 dev lo # on node-peer-b (internal ip)
```

## 9. restart ipsec daemon, start tunnel and test ping (on all peer nodes)

```sh
ipsec restart

ipsec up tunnel

ipsec status
```

## 10. test

```sh
ping 10.0.2.1 # on node-peer-a (10.0.1.1)
ping 10.0.1.1 # on node-peer-b (10.0.2.1)
```

## 11. clean
```sh
docker-machine rm node-main node-peer-a node-peer-b
```