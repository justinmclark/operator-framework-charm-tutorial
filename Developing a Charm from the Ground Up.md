# Developing a Charm from the Ground Up

The goal of this tutorial is to go through the detailed steps of developing a charm with no prior charm experience. Luckily, the Operator Framework makes this a much more pleasant experience.

**We will be developing a Kubernetes-Grafana charm in this tutorial.**

Here is the general flow this tutorial will follow:

1. Development setup
2. Dive into the documentation to understand what Grafana is and how to configure it
3. Design requirements for the Grafana charm
4. Understanding the boilerplate code created by `charmcraft init`
5. Get a bare-bones version of the Grafana charm up and running
6. Add the capability of relating a data source to the charm
7. Closing thoughts



## 1. Development Setup

> These steps are party taken from the Juju docs: https://juju.is/docs/microk8s-cloud

To get started, we will need a few key tools: `juju`, `microk8s`, and `charmcraft`.

```bash
snap install juju --classic
snap install microk8s --classic
snap install --beta charmcraft
```

Now, we need to get a Microk8s controller ready on Juju.

```bash
sudo usermod -a -G microk8s $USER
su - $USER
microk8s status --wait-ready  # waiting for installation to finish
microk8s.enable dns storage registry
juju bootstrap microk8s micro  # get the controller running
```

Lastly, we need the Python 3.5+ and the Operator Framework.

```bash
pip3 install ops
```

We're now all set for the rest of this tutorial.

## 2. Grafana Documentation

Before we can write a charm for some application, we need to understand it. Of course, this will not be an exhaustive coverage of Grafana, but it will be enough to get up and running.

[From the Grafana docs](https://grafana.com/docs/grafana/latest/getting-started/what-is-grafana/):

> Grafana is open source visualization and analytics software. It  allows you to query, visualize, alert on, and explore your metrics no  matter where they are stored. In plain English, it provides you with  tools to turn your time-series database (TSDB) data into beautiful  graphs and visualizations.

For our use case, Grafana is used as part of a logging, monitoring, alerting (LMA) stack but Grafana can actually be used in many different contexts. For example, [one team used Grafana to monitor beehives](https://grafana.com/blog/2020/05/21/grafanaconline-day-6-recap-the-power-of-tanka-and-a-peek-into-the-world-of-beehive-monitoring-with-grafana-dashboards/).

TODO: talk about prometheus

TODO: talk about provisioning

There are two files of interest:

- `grafana.ini` is the file where all the standard configuration options will be stored. [According to the docs](https://grafana.com/docs/grafana/latest/administration/configuration/#config-file-locations), this file is located at `/etc/grafana/grafana.ini`
- `datasources.yaml` is where we will provision new data sources. This file will be located at `<provisioning_path>/datasources/datasources.yaml`. Note that `<provisioning_path>` can be defined in `grafana.ini` but the default is `/etc/grafana/provisioning`.

## 3. Design Requirements

Let's discuss some key design considerations for this charm.

1. Grafana requires some sort of database to store dashboards and other essential information. [From the Grafana docs](https://grafana.com/docs/grafana/latest/administration/configuration/#database), we see that the default database is `sqlite3` but Grafana can also connect to `mysql` and `postgres` databases. Note that `sqlite3` will only be a viable option if we only have one running Grafana instance. If we wanted high-availability (HA), we would need a single `mysql` or `postgres` database that every Grafana instance could talk to.
2. We are developing a Kubernetes charm. This means Grafana will be running in a container (i.e. no persistent storage). What happens if our Grafana container crashes? Without persistent storage, our `sqlite3` database would disappear and we would have to start from scratch. This is a huge no-no. So, we will be covering how to ensure our `sqlite3` database lives beyond the life of any individual container.
3. In order for it to be useful, we will need to ensure data sources can be consumed by Grafana and the associated configuration changes trigger a restart of the Kubernetes pod. Note that Grafana requires its container to be restarted if any configuration changes take place, i.e. there is no command that will restart the application while keeping the container running.

## 4. Boilerplate

```bash
mkdir grafana-operator-tutorial
cd grafana-operator-tutorial
charmcraft init  # setup the directory to include all boilerplate code
```

Let's take a look at the directory structure:

```bash
$ tree .
.
├── actions.yaml
├── config.yaml
├── LICENSE
├── metadata.yaml
├── README.md
├── requirements-dev.txt
├── requirements.txt
├── run_tests
├── src
│   └── charm.py
└── tests
    ├── __init__.py
    └── test_charm.py

2 directories, 11 files
```

In this tutorial, our primary focus will be on `src/charm.py`, `tests/test_charm.py`, `metadata.yaml`, and `config.yaml`. It is worth exploring this boilerplate code to get an understanding of what a basic charm looks like.

Also, before continuing it would be beneficial to check out some of the existing documentation on the Operator Framework and Kubernetes charms.

TODO: link to Facundo's **Writing a Charm Using the Operator Framework** doc.

TODO: link to pod spec documentation

## 5. Bare-Bones Deployment

Let's get into the meat of this deployment. We will edit `config.yaml`, `metadata.yaml`, `src/charm.py`, and `tests/test_charm.py` and then test out a bare-bones Grafana deployment.

#### `config.yaml`

We will concern ourselves with the following configuration parameters:

* `advertised_port`: Where Grafana listens for incoming dashboard connections.
* `grafana_image_path`: The path to the Grafana container image. Docker path is used by default.
* `grafana_image_username`: If using a private container registry, this is where the username is specified.
* `grafana_image_password`: If using a private container registry, this is where the password is specified.
* `provisioning_path`: As noted in [Section 2](#grafana-documentation), we will define the provisioning path (using the Grafana default, but allowing other options).

```yaml
# config.yaml
options:
    advertised_port:
        description: The port Grafana will be listening on.
        type: int
        default: 3000
    grafana_image_path:
        type: string
        description: |
            The location of the image to use, e.g. "registry.example.com/grafana:v1". 
            This setting is required.
        default: 'grafana/grafana:latest'
    grafana_image_username:
        type: string
        description: |
            The username for accessing the registry specified by grafana_image_path.
        default: ''
    grafana_image_password:
        type: string
        description: |
            The password associated with grafana_image_username for
            accessing the registry specified by grafana_image_path.
        default: ''
    provisioning_path:
        type: string
        description: |
            The location of all the provisioning files used by Grafana.
        default: /etc/grafana/provisioning
```

#### `metadata.yaml`

Now we will define our basic `metadata.yaml` file. As mentioned in [Section 3](#design-requirements), we will need to define persistent storage so our `sqlite3` database exists beyond the lifespan of the container. [According to the Grafana docs](https://grafana.com/docs/grafana/latest/installation/debian/#package-details):

> The default configuration specifies an sqlite3 db at `/var/lib/grafana/grafana.db`

So, we need to make sure we have persistent storage for `/var/lib/grafana`. [Check out the Juju docs for more information on how persistent storage works with Kubernetes](https://juju.is/docs/persistent-storage-and-kubernetes).

```yaml
name: grafana
summary: Data vizualization for the LMA operator stack
maintainers:
    - Alan Turning <alan.turing@example.com>
description: This charm will manage Grafana.
series:
    - kubernetes
storage:
    sqlitedb:
        type: filesystem
        location: /var/lib/grafana
```

#### `src/charm.py`

Don't be intimidated by the ~100 lines of code here. A pretty hefty chunk of this is just a dictionary definition. See if you can figure out what's going on:

```python
#! /usr/bin/env python3
# -*- coding: utf-8 -*-

import logging
import textwrap

from ops.charm import CharmBase
from ops.framework import StoredState
from ops.main import main
from ops.model import ActiveStatus, MaintenanceStatus

log = logging.getLogger()


class GrafanaK8s(CharmBase):
    """Charm to run Grafana on Kubernetes.

    Developers of this charm should be aware of the Grafana provisioning docs:
    https://grafana.com/docs/grafana/latest/administration/provisioning/
    """
    
    # this will be used in the future and is a core part of
    # Operator Framework charms
    store = StoredState()

    def __init__(self, *args):
        log.debug('Initializing charm.')
        super().__init__(*args)

        # basic event observations
        # e.g. when Juju fires the "config_changed" event, then
        # 	   our "on_config_changed() handler will be triggered"
        self.framework.observe(self.on.config_changed, self.on_config_changed)

    def on_config_changed(self, event):
        self.configure_pod()

    def _build_pod_spec(self):
        """Builds the pod spec based on available info in datastore`."""
        
        config = self.model.config

        # set image details based on what is defined in the charm configuation
        image_details = {
            'imagePath': config['grafana_image_path']
        }
        if config['grafana_image_username']:
            image_details['username'] = config['grafana_image_username']
            image_details['password'] = config['grafana_image_password']

        spec = {
            'version': 3,
            'containers': [{
                'name': self.app.name,  # self.app.name is defined in metadata.yaml
                'imageDetails': image_details,
                'ports': [{
                    'containerPort': config['advertised_port'],
                    'protocol': 'TCP'
                }],
                
                # this where we define any files necessary for configuration
                # Juju gives developers a nice way of directly defining what
                # the contents of files should be.
                
                # Note that "volumeConfig" is new in pod-spec v3 and is a
                # replacement for "files"
                'volumeConfig': [{
                    'name': 'config',
                    'mountPath': '/etc/grafana',
                    'files': [{
                        'path': 'grafana.ini',
                        
                        # this is a very basic configuration file with
                        # some hard coded options for demonstration
                        # consider adding this kind of information in
                        # `config.yaml` in production charms
                        'content': textwrap.dedent("""
                            [paths]
                            provisioning = {}
                            
                            [database]
                            type = sqlite3

                            [log]
                            mode = file
                            level = info
                            """.format(config['provisioning_path']))
                    }],
                }],
            }]
        }

        return spec

    def configure_pod(self):
        """Set Juju / Kubernetes pod spec built from `_build_pod_spec()`."""

        # even though we won't be supporting peer relations in this tutorial
        # it is best practice to check whether this unit is the leader unit
        if not self.unit.is_leader():
            log.debug('Unit is not leader. Cannot set pod spec.')
            return
        
        # setting pod spec and associated logging
        self.unit.status = MaintenanceStatus('Building pod spec.')
        log.debug('Building pod spec.')

        pod_spec = self._build_pod_spec()
        self.model.pod.set_spec(pod_spec)

        self.unit.status = ActiveStatus()
        log.debug('Pod spec set successfully.')


if __name__ == '__main__':
    main(GrafanaK8s)
```

#### `tests/test_charm.py`

As with any software project, unit tests are a key part of development. The Operator Framework includes a testing Harness which gives charm developers a very nice way of unit testing charms written with the Operator Framework.

We will add one unit test as an example. _To test your understanding of the test harness, try adding more tests yourself! You can run them with `./run_tests`_.

```python
# See LICENSE file for licensing details.

import unittest

from ops.testing import Harness
from charm import GrafanaK8s

BASE_CONFIG = {
    'advertised_port': 3000,
    'grafana_image_path': 'grafana/grafana:latest',
    'grafana_image_username': '',
    'grafana_image_password': '',
    'provisioning_path': '/etc/grafana/provisioning',
}


class TestCharm(unittest.TestCase):
    
    def setUp(self):
        self.harness = Harness(GrafanaK8s)
        self.addCleanup(self.harness.cleanup)
        self.harness.begin()

    def test__image_details_in_pod_spec(self):
        """Test whether image details come from config properly."""

        # basic setup
        self.harness.set_leader(True)  # required so we can set pod spec
        self.harness.update_config(BASE_CONFIG)

        # emit the "config_changed" event and get pod spec
        self.harness.charm.on.config_changed.emit()
        pod_spec, _ = self.harness.get_pod_spec()

        # test image details
        expected_image_details = {
            'imagePath': 'grafana/grafana:latest',
        }

        self.assertEqual(expected_image_details,
                         pod_spec['containers'][0]['imageDetails'])
```

#### Time to Deploy!

Here are the steps to deploy your newly built charm:

``` bash
# still in the grafana-operator-tutorial directory
juju add-model lma  # this is the model in which we'll deploy the new charm
juju create-storage-pool operator-storage \
	kubernetes storage-class=microk8s-hostpath  # so persistent storage works as expected
charmcraft build  # build the charm
juju deploy ./grafana.charm
watch -c juju status --color  # wait until everything has settled
```

`juju status` should now look something like:

```
Model  Controller  Cloud/Region        Version    SLA          Timestamp
lma    mk8s        microk8s/localhost  2.9-beta1  unsupported  16:39:21-04:00

App      Version         Status  Scale  Charm    Store  Rev  OS          Address         Message
grafana  grafana:latest  active      1  grafana  local    0  kubernetes  10.152.183.115  

Unit        Workload  Agent  Address      Ports     Message
grafana/0*  active    idle   10.1.185.50  3000/TCP  
```

Now, lets go through some other checks to make sure everything worked the way it should have.

**Have all of our K8s resources started as expected?**

```bash
$ microk8s.kubectl get all -n lma
NAME                                 READY   STATUS    RESTARTS   AGE
pod/modeloperator-7f45769cbb-nw9f5   1/1     Running   0          74m
pod/grafana-operator-0               1/1     Running   0          74m
pod/grafana-0                        1/1     Running   0          74m

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
service/modeloperator       ClusterIP   10.152.183.13    <none>        17071/TCP   74m
service/grafana-operator    ClusterIP   10.152.183.47    <none>        30666/TCP   74m
service/grafana             ClusterIP   10.152.183.115   <none>        3000/TCP    74m
service/grafana-endpoints   ClusterIP   None             <none>        <none>      74m

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/modeloperator   1/1     1            1           74m

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/modeloperator-7f45769cbb   1         1         1       74m

NAME                                READY   AGE
statefulset.apps/grafana-operator   1/1     74m
statefulset.apps/grafana            1/1     74m
```

**Did our `grafana.ini` file get set correctly?**

```bash
$ microk8s.kubectl exec -n lma grafana-0 -- cat /etc/grafana/grafana.ini

[paths]
provisioning = /etc/grafana/provisioning

[database]
type = sqlite3

[log]
mode = file
level = info
```

**Can we access the Grafana dashboard?**

Using the application IP address from `juju status` (`10.152.183.115` in our case), navigate to `http://10.152.183.115:3000` in your browser (changing the IP address to match your own deployment).

If everything is working properly, you should see a log-in screen that looks like this:

![](/home/justin/Pictures/Screenshot from 2020-09-24 16-55-06.png)

> Note that `username` and `password` are both "admin" by default.

You will be prompted to change the password, but feel free to skip this step. There's not much to do since we have no data sources, but hey, we have our basic charm up and running!

## 6. Add a Data Source



## 7. Closing Thoughts



