# Injecting Proxy Settings into PKS
## Modify kubo release
Release version reference:
* PKS 1.3.6 uses kubo 0.25.12 and docker 35.1.0
* PKS 1.4.0 uses kubo 0.31.0 and docker 35.1.0
Fork https://github.com/pivotal-cf/pks-kubo-release and checkout appropriate branch.
    cd ~
    git clone https://github.com/bkirkware/pks-kubo-release.git
    cd pks-kubo-release
    git checkout v0.25.12
`vi jobs/kubelet/templates/bin/kubelet_ctl.erb`
    
    export HTTP_PROXY="<proxy url>"
    export HTTPS_PROXY="squid.kirklab.io:3128"
    export http_proxy="squid.kirklab.io:3128"
    export https_proxy="squid.kirklab.io:3128"
    export no_proxy="localhost,127.0.0.1,harbor.pks.kirklab.io,169.254.169.254,local,svc.cluster.local,192.168.192.0/20,172.16.0.0/16,internal,master.cfcr.internal"
Create and upload release
    
    bosh create-release --force --tarball /tmp/pks-kubo-release-modified.tgz
    bosh -e kirklab upload-release /tmp/pks-kubo-release-modified.tgz --fix
Copy the manifest and update with modified release information
    sudo cp /var/tempest/workspaces/default/deployments/pivotal-container-service-d2b7297f54526b31869b.yml /tmp/pks-proxy.yml
    sudo chown ubuntu:ubuntu /tmp/pks-proxy.yml
    vi /tmp/pks-proxy.yml
    :%s/0.25.12/0.25.0+dev.1/g
## Modify docker release
Pull down docker-boshrelease and docker swarm code. Checkout branch v35.1.0 of docker-boshrelease.
    cd ~
    https://github.com/cloudfoundry-incubator/docker-boshrelease.git
    cd docker-boshrelease
    git checkout v35.1.0
    cd src/github.com/docker
    rm -r swarm
    git clone https://github.com/docker/swarm.git
    cd ../../..
`vi jobs/docker/templates/bin/job_properties.sh.erb`
    export HTTP_PROXY="squid.kirklab.io:3128"
    export HTTPS_PROXY="squid.kirklab.io:3128"
    export http_proxy="squid.kirklab.io:3128"
    export https_proxy="squid.kirklab.io:3128"
    export no_proxy="localhost,127.0.0.1,harbor.pks.kirklab.io,169.254.169.254,local,svc.cluster.local,192.168.192.0/20,172.16.0.0/16,internal,master.cfcr.internal"
Create and upload release
    bosh create-release --force --tarball /tmp/docker-modified.tgz
    bosh -e kirklab upload-release /tmp/docker-modified.tgz --fix
Update manifest with modified release information
    vi /tmp/pks-proxy.yml
    :%s/35.1.0/35.1.0+dev.1/g
## Apply these changes
    bosh -e kirklab -d pivotal-container-service-d2b7297f54526b31869b deploy /tmp/pks-proxy.yml
## Create a test cluster
    pks login -a api.pks.kirklab.io --ca-cert ~/Work/homelab/ca/pks.crt -u bkirkland
    pks create-cluster tibanna --external-hostname tibanna.kirklab.io --plan small
    watch pks cluster tibanna
