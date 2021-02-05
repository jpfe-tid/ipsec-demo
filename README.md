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
  node-a

docker-machine create \
  --driver amazonec2 \
  --amazonec2-instance-type t3.small \
  --amazonec2-region ${AWS_REGION} \
  --amazonec2-root-size 30 \
  --amazonec2-ami ${AMI_ID} \
  node-b

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

## 2. ssh and install packages (on each node)

```sh
docker-machine ssh node-a
docker-machine ssh node-b
```

```sh
sudo apt update

sudo apt install \
  iproute2 \
  strongswan \
  strongswan-pki \
  ipsec-tools
```

## 3. generate ca (optional, provided for convenience)

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

# copy ca-cert.pem to all nodes
```

## 4. generate certs (on each node)

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

mkdir -p pki/certs
ipsec pki \
  --pub \
  --in ${HOSTNAME}-key.pem \
  --type rsa | \
  ipsec pki \
    --issue \
    --lifetime 1825 \
    --cacert ca-cert.pem \
    --cakey ca-key.pem \
    --dn "CN=${IP}" \
    --san "${IP}" \
    --flag serverAuth \
    --flag ikeIntermediate \
    --outform pem \
      > ${HOSTNAME}-cert.pem
```

## 5. copy certs and keys (on each node)

```sh
cp ca-cert.pem /etc/ipsec.d/cacerts/ca-cert.pem

cp ${HOSTNAME}-cert.pem /etc/ipsec.d/certs/${HOSTNAME}-cert.pem

cp ${HOSTNAME}-key.pem /etc/ipsec.d/private/${HOSTNAME}-key.pem

ls -lah \
  /etc/ipsec.d/cacerts \
  /etc/ipsec.d/certs \
  /etc/ipsec.d/private
```

## 6. copy conf (on each node)

```sh
export LEFT_IP='172.31.1.101' # node-a IP
export LEFT_HOSTNAME='node-a'
export RIGHT_IP='172.31.1.102' # node-b IP
export RIGHT_HOSTNAME='node-b'
export HOSTNAME
```

```sh
cat ipsec.conf| envsubst > /etc/ipsec.conf

cat ipsec.secrets| envsubst > /etc/ipsec.secrets

ls -lah \
  /etc/ipsec.secrets \
  /etc/ipsec.conf
```

## 7. restart ipsec daemon, start tunnel and test ping (on each node)

```sh
ip a a 10.0.1.1 dev lo # on node-a (internal ip)
ip a a 10.0.2.1 dev lo # on node-b (internal ip)

ipsec restart

ipsec up tunnel

ipsec status

ping 10.0.2.1 # on node-a (10.0.1.1)
```