# metadataproxy

The metadataproxy is used to allow containers to acquire IAM roles.

## Installation

From inside of the repo run the following commands:

```bash
mkdir -p /srv/metadataproxy
cd /srv/metadataproxy
virtualenv venv
source venv/bin/activate
pip install metadataproxy
deactivate
```

## Configuration

### Modes of operation

See [the settings file](https://github.com/lyft/metadataproxy/blob/master/metadataproxy/settings.py)
for specific configuration options.

The metadataproxy has two basic modes of operation:

1. Running in AWS where it simply proxies most routes to the real metadata
   service.
2. Running outside of AWS where it mocks out most routes.

To enable mocking, use the environment variable:

```
export MOCK_API=true
```

### AWS credentials

metadataproxy relies on boto configuration for its AWS credentials. If metadata
IAM credentials are available, it will use this. Otherwise, you'll need to use
.aws/credentials, .boto, or environment variables to specify the IAM
credentials before the service is started.

### Role assumption

For IAM routes, the metadataproxy will use STS to assume roles for containers.
To do so it takes the incoming IP address of metadata requests and finds the
running docker container associated with the IP address. It uses the value of
the container's `IAM_ROLE` environment variable as the role it will assume. It
then assumes the role and gives back STS credentials in the metadata response.

STS-attained credentials are cached and automatically rotated as they expire.

#### Container-specific roles

To specify the role of a container, simply launch it with the `IAM_ROLE`
environment variable set to the IAM role you wish the container to run with.

```shell
docker run -e IAM_ROLE=my-role ubuntu:14.04
```

#### Default Roles

When no role is matched, `metadataproxy` will use the role specified in the 
`DEFAULT_ROLE` `metadataproxy` environment variable. If no DEFAULT_ROLE is
specified as a fallback, then your docker container without an `IAM_ROLE`
environment variable will fail to retrieve credentials.

The `DEFAULT_ROLE` environment variable can be specified locally:

```shell
export DEFAULT_ROLE=my-default-role
```

Or can be specified via docker env:

```shell
docker run \
       -e DEFAULT_ROLE=my-default-role \
       ...
```

#### Role Formats

The following are all supported formats for specifying roles:

- By Role:

    ```shell
    IAM_ROLE=my-role
    ```

- By Role@AccountId

    ```shell
    IAM_ROLE=my-role@012345678910
    ```

- By ARN:

    ```shell
    IAM_ROLE=arn:aws:iam::012345678910:role/my-role
    ```

### Role structure

A useful way to deploy this metadataproxy is with a two-tier role
structure:

1.  The first tier is the EC2 service role for the instances running
    your containers.  Call it `DockerHostRole`.  Your instances must
    be launched with a policy that assigns this role.

2.  The second tier is the role that each container will use.  These
    roles must trust your own account ("Role for Cross-Account
    Access" in AWS terms).  Call it `ContainerRole1`.

3.  metadataproxy needs to query and assume the container role.  So
    the `DockerHostRole` policy must permit this for each container
    role.  For example:
    ```
    "Statement": [ {
        "Effect": "Allow",
        "Action": [
            "iam:GetRole",
            "sts:AssumeRole"
        ],
        "Resource": [
            "arn:aws:iam::012345678901:role/ContainerRole1",
            "arn:aws:iam::012345678901:role/ContainerRole2"
        ]
    } ]
    ```

4. Now customize `ContainerRole1` & friends as you like

Note: The `ContainerRole1` role should have a trust relationship that allows it to be assumed by the `user` which is associated to the host machine running the `sts:AssumeRole` command.  An example trust relationship for `ContainRole1` may look like:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::012345678901:root",
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Routing container traffic to metadataproxy

Using iptables, we can forward traffic meant to 169.254.169.254 from docker0 to
the metadataproxy. The following example assumes the metadataproxy is run on
the host, and not in a container:

```
/sbin/iptables \
  --append PREROUTING \
  --destination 169.254.169.254 \
  --protocol tcp \
  --dport 80 \
  --in-interface docker0 \
  --jump DNAT \
  --table nat \
  --to-destination 127.0.0.1:8000 \
  --wait
```

If you'd like to start the metadataproxy in a container, it's recommended to
use host-only networking. Also, it's necessary to volume mount in the docker
socket, as metadataproxy must be able to interact with docker.

Be aware that non-host-mode containers will not be able to contact
127.0.0.1 in the host network stack.  As an alternative, you can use
the meta-data service to find the local address.  In this case, you
probably want to restrict proxy access to the docker0 interface!

```
LOCAL_IPV4=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)

/sbin/iptables \
  --append PREROUTING \
  --destination 169.254.169.254 \
  --protocol tcp \
  --dport 80 \
  --in-interface docker0 \
  --jump DNAT \
  --table nat \
  --to-destination $LOCAL_IPV4:8000 \
  --wait

/sbin/iptables \
  --wait \
  --insert INPUT 1 \
  --protocol tcp \
  --dport 80 \
  \! \
  --in-interface docker0 \
  --jump DROP
```

## Run metadataproxy without docker

In the following we assume _my\_config_ is a bash file with exports for all of
the necessary settings discussed in the configuration section.

```
source my_config
cd /srv/metadataproxy
source venv/bin/activate
gunicorn metadataproxy:app --workers=2 -k gevent
```

## Run metadataproxy with docker

For production purposes, you'll want to kick up a container to run.
You can build one with the included Dockerfile.  To run, do something like:
```bash
docker run --net=host \
    -v /var/run/docker.sock:/var/run/docker.sock \
    metadataproxy
```

### gunicorn settings

The following environment variables can be set to configure gunicorn (defaults
are set in the examples):

```
# Change the IP address the gunicorn worker is listening on. You likely want to
# leave this as the default
HOST=0.0.0.0

# Change the port the gunicorn worker is listening on.
PORT=8000

# Change the number of worker processes gunicorn will run with. The default is
# 1, which is likely enough since metadataproxy is using gevent and its work is
# completely IO bound. Increasing the number of workers will likely make your
# in-memory cache less efficient
WORKERS=1

# Enable debug mode (you should not do this in production as it will leak IAM
# credentials into your logs)
DEBUG=False
```

## Contributing

### Code of conduct

This project is governed by [Lyft's code of
conduct](https://github.com/lyft/code-of-conduct).
All contributors and participants agree to abide by its terms.

### Sign the Contributor License Agreement (CLA)

We require a CLA for code contributions, so before we can accept a pull request
we need to have a signed CLA. Please [visit our CLA
service](https://oss.lyft.com/cla)
follow the instructions to sign the CLA.

### File issues in Github

In general all enhancements or bugs should be tracked via github issues before
PRs are submitted. We don't require them, but it'll help us plan and track.

When submitting bugs through issues, please try to be as descriptive as
possible. It'll make it easier and quicker for everyone if the developers can
easily reproduce your bug.

### Submit pull requests

Our only method of accepting code changes is through github pull requests.
