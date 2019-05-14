# openshift-container-signing
# How to deploy
Verify vars are correct by overridding them or changing the default values.  I.e. change the domain name for the registry.

Then run the playbook as follows, or as appropriate for your environment.
ansible-playbook -kKbu sysadmin openshift-container-signing.yml

# Sign image (from basebox)
## List images in registry
curl -X GET http://registry.example.com:5000/v2/_catalog

## List valid tags for an image
curl -X GET http://registry.example.com:5000/v2/rhscl/postgresql-96-rhel7/tags/list

## Use scopeo to copy signed image to registry
skopeo copy --sign-by testing@example.com --src-tls-verify=false --dest-tls-verify=false docker://registry.example.com:5000/rhscl/postgresql-96-rhel7:1-32 docker://registry.example.com:5000/rhscl/postgresql-96-rhel7:signed

## Verify image digest matches
skopeo inspect --tls-verify=false docker://registry.example.com:5000/rhscl/postgresql-96-rhel7:signed | grep Digest
ll /var/lib/atomic/sigstore/rhscl/
curl http://registry.example.com/sigstore/rhscl/

# Test deployment
## Attempt pull from appnode
atomic pull registry.example.com:5000/rhscl/postgresql-96-rhel7:1-32  #should fail
atomic pull registry.example.com:5000/rhscl/postgresql-96-rhel7:signed  #should successfully pull

## Create new project
oc new-project signed-images

## Create new app
oc new-app --insecure-registry=true --docker-image=registry.example.com:5000/rhscl/postgresql-96-rhel7:1-32 --name=signed-pgsql

# Notes
## Clean up images that are not tagged
docker rmi $(docker images --filter "dangling=true" -q --no-trunc)
