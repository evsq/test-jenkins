docker run -d -p 50000:50000 -p 8080:8080 -v $HOME/jenkins_home:/var/jenkins_home --name jenkins jenkins/jenkins --restart=always


mkdir certs

openssl req \
  -newkey rsa:4096 -nodes -sha256 -x509 -days 3650 \
  -subj "/C=RU/ST=Russia/L=Moscow/O=test/OU=test/CN=testregistry.com" \
  -keyout certs/testregistry.com.key \
  -out certs/testregistry.com.crt

mkdir -p /etc/docker/certs.d/testregistry.com/

cp certs/testregistry.com.crt /etc/docker/certs.d/testregistry.com/

mkdir auth

apt-get install apache2-utils -y

htpasswd -Bbn testuser testpassword > auth/htpasswd

docker run -d \
  --restart=always \
  --name registry \
  -v "$(pwd)"/certs:/certs \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/testregistry.com.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/testregistry.com.key \
  -v "$(pwd)"/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -p 443:443 \
  registry:2

mkdir test-registry && cd test-registry

cat <<EOF > Dockerfile
FROM ubuntu:18.04
RUN apt-get update && \
 apt-get install -y python
EOF

docker build -t ubuntu-python .

docker login testregistry.com --username testuser --password testpassword

docker tag ubuntu-python testregistry.com/ubuntu-python

docker push testregistry.com/ubuntu-python

docker rmi ubuntu:18.04 ubuntu-python testregistry.com/ubuntu-python

docker pull testregistry.com/ubuntu-python

