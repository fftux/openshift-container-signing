# openshift-container-signing
## Assumptions
This repository assumes that you plan to deploy an external registry, where you will sign all container images in the registry with your own GPG key.  This may be used for an offline installation, where this registry holds all of the necessary containers images for installation.  Alternatively, it may be simply used for custom images.

Additionally, this repo assumes that the registry server is used as the control node.  This, of course, can easily be changed, but that is how the roles are currently written.  

## How to deploy
Verify vars are correct by overridding them or changing the default values.  I.e. change the domain name for the registry. Registry must also be defined in the ansible hosts file.

Pull down the code.
```sh
git clone -b $BRANCH https://github.com/chopskxw/openshift-container-signing.git [$WORKING_DIR]
```

Then run the playbook as follows, or as appropriate for your environment.
```sh
ansible-playbook openshift-container-signing.yml
```

## Sign image (from registry)
### List images in registry
```sh
curl -X GET http://registry.example.com:5000/v2/_catalog
```

### List valid tags for an image
```sh
curl -X GET http://registry.example.com:5000/v2/rhscl/postgresql-96-rhel7/tags/list
```

### Use scopeo to copy signed image to registry
```sh
skopeo copy --sign-by testing@example.com --src-tls-verify=false --dest-tls-verify=false \
docker://registry.example.com:5000/rhscl/postgresql-96-rhel7:1-32 \
docker://registry.example.com:5000/rhscl/postgresql-96-rhel7:signed
```
Or, if pulling from Red Hat.
```sh
skopeo copy --remove-signatures --sign-by testing@example.com --dest-tls-verify=false \
docker://registry.redhat.io/rhel7/etcd \
docker://registry.williamscomplex.local:5000/signed/etcd
```

### Verify image digest matches
```sh
skopeo inspect --tls-verify=false \
docker://registry.example.com:5000/rhscl/postgresql-96-rhel7:signed | grep Digest
ll /var/lib/atomic/sigstore/rhscl/
curl http://registry.example.com/sigstore/rhscl/
```

## Test deployment
### Attempt pull from appnode
```sh
atomic pull registry.example.com:5000/rhscl/postgresql-96-rhel7:1-32  #should fail
atomic pull registry.example.com:5000/rhscl/postgresql-96-rhel7:signed  #should successfully pull
```

### Create new project
```sh
oc new-project signed-images
```

### Create new app
```sh
oc new-app --insecure-registry=true \
--docker-image=registry.example.com:5000/rhscl/postgresql-96-rhel7:1-32 --name=signed-pgsql
```

## Notes
### Clean up images that are not tagged
Use this to clean up cached images on nodes.  This will insure that all policies are adhered to.
```sh
docker rmi $(docker images --filter "dangling=true" -q --no-trunc)
```

### Policies for online registries
If the cluster is online, the following commands will setup policies for online registries.  The Red Hat registries can be configured for signature verification.
```sh
## policy for docker.io
atomic --assumeyes trust add docker.io --type insecureAcceptAnything
## policy for registry.access.redhat.com
atomic --assumeyes trust add --sigstoretype web \
--sigstore https://access.redhat.com/webassets/docker/content/sigstore \
--pubkeys /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release registry.access.redhat.com
## policy for registry.redhat.io
atomic --assumeyes trust add --sigstoretype web \
--sigstore https://access.redhat.com/webassets/docker/content/sigstore \
--pubkeys /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release registry.redhat.io
```
