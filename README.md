Andock
======

An Ansible role that can be used as a dependency in other roles, to produce Docker images.


Motivation
----------

- Ansible playbooks are generic recipies for how to put a machine in a specific state.

- Dockerfiles are recipies for how to put a Docker continer in a specific state.

If we buy into the appeal of Docker (or [solutions like it](https://github.com/coreos/rkt)),
the quickest, most natural way to get started is to describe what the container should contain
in a Dockerfile.

On the other hand, if we write our service recipies as Ansible playbooks rather than Dockerfiles,
then they are compatible as is for deployment to non-Docker hosts too
(VM:s, physical servers, your local computer, ...)


Alternatives
------------

[ansible-docker-base](https://github.com/ansible/ansible-docker-base)
accomplishes pretty much the same thing, by including Ansible and the playbook
inside the container. If this is not a deal-breaker, then ansible-docker-base
is probably a simpler approach. A goal with Andock is to produce a container
that has never seen the playbook it was built from.

Example use-case where running Ansible from outside the container is a benefit:
Your playbook uses a secret key to produce a public signature. You want to
include the signature in the container, but not the secret.

You can probably accomplish some separation towards the same goal with
ansible-docker-base too, if you temporarily mount the playbook as a
[Docker volume](http://docs.docker.com/userguide/dockervolumes/) but you do
put a lot more trust in all the moving parts involved: The secret key has existed
in the Docker image you produce and in the worst case, some component you rely on
might have left temp files behind, etc.


Workflow
------------

### Input to Andock

- One "service playbook", which describe the service(s) to produce, typically
in a Docker-agnostic way.

- One "Docker playbook", where one or more roles declare Andock as a dependency,
where each declaration includes the role to perform in the service playbook
and the name of the Docker image to produce.

The Docker playbook is typically performed from localhost targeting the Docker
host machine - perhaps localhost in a dev scenario.

The service playbook is performed from the docker host, targeting a temporary
docker container.


Requirements
------------

- A `Docker host` machine with Docker and Ansible installed (localhost or remote)


Role Variables
--------------

- andock_image
  The name of the Docker image to be produced

- andock_role
  A role in the service playbook to be performed in the Docker container

- andock_parent_image
  An existing Docker image to use as starting point for the container.
  Assumed to be either [phusion/baseimage](https://github.com/phusion/baseimage-docker)
  or a derivative of it - a common pattern is to refer to a previously genereated
  Andock-generated image (ensure that it is unconditionally "generated previously",
  by declaring the role that produces it as a dependency).
  phusion/basimage enables us to let Ansible log in from the outside over SSH.

- andock_identity_file
  A keypair to use for SSH access. The public key will be installed in
  /root/.ssh/authorized_keys in the resulting Docker image

- andock_docker_hostname
  The hostname or ip where Docker container port mappings are forwarded to.
  Ansible will SSH to this host on the port that container's port 22 is forwarded to.
  Default: "localhost"

- andock_cmd_docker
  The docker command. Default: `/usr/local/bin/docker`

- andock_cmd_ansible
  The docker command. Default: `/usr/local/bin/ansible-playbook`

- andock_virtualenv
  Optionally activate a virtualenv before executing the shell commands
  that Andock depend on.


Example Playbook
----------------

### Example: Create an Nginx Docker image based on baseimage and an Nginx Ansible play

You must provide an SSH keypair so that Ansible can log in to an instance of the
parent image and modify it. Andock will add the public key to ~/.ssh/authorized_keys
of the image. Create a keypair with `ssh-keygen`.

```
# in roles/andock-examples-nginx/meta/main.yml
dependencies:
  - role: andock
    andock_image: "andock-examples/nginx"
    andock_role: "/path/to/playbook/nginx"
    andock_parent_image: "phusion/baseimage"
    andock_identity_file: "/path/to/ssh-public_key"
```

### Example: Create specialized Nginx Docker images based on the generic Nginx image created above

Suppose we need multiple Docker images to satisfy different use-cases.
In this example: one barebones Nginx-image, another one that supports PHP,
a third one without PHP but with Node.js

We could start both specialized images from phusion/baseimage
and let the Ansible playbook build them from scratch in isolation.
Alternatively, by specifying the base Nginx image as the parent,
we can get Docker to store the specialized versions more efficiently,
as diffs of the base image - and we can ensure that Ansible has less
work to do: the vanilla Nginx parts only have to be performed once.

We already added the public ssh key to the public image. If Ansible has access
to the keypair implicitly, by it residing in ~/.ssh/, then andock_ssh_key
can be omitted here. If not, specify andock_ssh_key here too.

```
# in roles/andock-examples-nginx-with-php/meta/main.yml
dependencies:
  # ensure that the parent image is built
  - role: andock-examples-nginx

  # let andock build dependent image
  - role: andock
    andock_image: "andock-examples/nginx-with-php"
    andock_role: "/path/to/playbook/nginx-php"
    andock_parent_image: "andock-examples/nginx"
```

```
# in roles/andock-examples-nginx-with-node/meta/main.yml
dependencies:
  # ensure that the parent image is built
  - role: andock-examples-nginx

  # let andock build dependent image
  - role: andock
    andock_image: "andock-examples/nginx-with-node"
    andock_role: "/path/to/playbook/nginx-node"
    andock_parent_image: "andock-examples/nginx"
```


Author Information
------------------

Tom Sundström (office@tomsun.ax)
