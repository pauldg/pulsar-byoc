# Pulsar endpoint for Galaxy with SLURM or HTCondor

This playbook sets up a Pulsar endpoint that a Galaxy server can send jobs to. Pulsar runs on one host. The jobs run on one or more worker hosts, through SLURM or HTCondor. Pulsar receives jobs from Galaxy over RabbitMQ or over [pulsar-relay](https://github.com/galaxyproject/pulsar-relay). With the relay, Galaxy and Pulsar both connect out to the relay over HTTPS, so the Pulsar site needs no open ports for messages.

The playbook adds no new roles. It combines existing community roles and sets their variables from the inventory:

| Part | Role |
| --- | --- |
| Pulsar | [galaxyproject.pulsar](https://github.com/galaxyproject/ansible-pulsar) |
| SLURM | [galaxyproject.slurm](https://github.com/galaxyproject/ansible-slurm) |
| HTCondor | [grycap.htcondor](https://github.com/usegalaxy-eu/ansible-htcondor-grycap) (usegalaxy-eu fork) |
| Shared directory | [geerlingguy.nfs](https://github.com/geerlingguy/ansible-role-nfs) |
| Reference data and containers | [galaxyproject.cvmfs](https://github.com/galaxyproject/ansible-cvmfs), [usegalaxy_eu.apptainer](https://github.com/usegalaxy-eu/ansible-apptainer) |

It was started at the ELIXIR BioHackathon Europe 2026, project 1 "Bridging ELIXIR nodes for collaborative and scalable BYOC in Galaxy and TREs". The aim is that de.NBI SimpleVM, SURF Research Cloud and other cloud providers can call it with their own inventory to build a "bring your own compute" endpoint.

## How it works

The Pulsar host exports a shared directory over NFS. The workers mount it. Pulsar stages each job's files there, so the workers can read the inputs and write the outputs. The Pulsar host also runs the scheduler's controller: `slurmctld` for SLURM, or the HTCondor central manager and submit role. The workers run `slurmd` or the HTCondor execute role, with CVMFS and Apptainer for tools in containers.

Two variables choose the setup:

- `pulsar_scheduler`: `slurm` or `htcondor`
- `pulsar_connection`: `mq` or `relay`

With the relay, `pulsar_relay_placement` chooses which relay to use:

- `central` (default) uses usegalaxy.eu's relay at `stargate.galaxyproject.eu`. The account is `byoc_<id>`, where `<id>` is the user's encoded Galaxy user id (`pulsar_galaxy_user_id`, shown by `/api/users/current`). The user chooses the password. usegalaxy.eu creates the account after it merges the user's BYOC PR.
- `own` uses a relay the user runs, for example the [SURF Research Cloud relay component](https://github.com/mirelaminkova/pulsar-relay-src). Set `pulsar_relay_url`, `pulsar_relay_username` and `pulsar_relay_password`.

The Pulsar job manager is called `production` for RabbitMQ and `_default_` for the relay. These are the names usegalaxy.eu's runners expect. Override `pulsar_manager_name` for another Galaxy.

A host can be in both the `pulsar` and `workers` groups. That gives a single-VM endpoint for testing.

## Connection summary

At the end of the run the playbook writes `/opt/pulsar/byoc-connection.yml` on the Pulsar host and prints it. It lists the values the user enters in the usegalaxy.eu BYOC form (*User > Preferences > Manage information > byoc_pulsar*): the transport, the relay or RabbitMQ account, the staging directory and the CPU and memory limits of the largest worker. It holds no passwords. A provider such as SimpleVM can show this file to the user.

With the relay, the playbook also checks that Pulsar has connected. It signs in to the relay and waits for Pulsar's capability topic (`pulsar_capabilities`). On the central relay this only works after usegalaxy.eu has created the account, so a first run reports "relay login failed" in the summary instead of stopping. Run the playbook again after the PR is merged to confirm the connection.

## Usage

```bash
ansible-galaxy install -r requirements.yml
cp inventory.example.ini inventory.ini        # add your hosts
cp secrets.example.yml secrets.yml            # add the queue URL or relay password
ansible-playbook site.yml -e @secrets.yml -e pulsar_scheduler=htcondor \
  -e pulsar_connection=relay -e pulsar_galaxy_user_id=YOUR_ENCODED_ID
```

Set `pulsar_nfs_network` to the subnet of your hosts. The other defaults are in `group_vars/all/`, one file for each topic.

## Galaxy side

Galaxy sends jobs to this endpoint with the Pulsar message-queue runner. For the relay (Galaxy 26.0 or newer):

```yaml
runners:
  pulsar_relay:
    load: galaxy.jobs.runners.pulsar:PulsarMQJobRunner
    relay_url: https://RELAY_HOST
    relay_username: RELAY_USER
    relay_password: RELAY_PASSWORD
    relay_topic_prefix: PREFIX
    galaxy_url: https://GALAXY_HOST

execution:
  environments:
    byoc_pulsar:
      runner: pulsar_relay
      jobs_directory: /data/share/staging
      manager: _default_
      dependency_resolution: remote
      default_file_action: remote_transfer
```

For RabbitMQ, use `amqp_url` on the runner instead of the `relay_*` settings. Pulsar must be able to reach `galaxy_url` over HTTPS in both cases, because files go directly between Pulsar and Galaxy.

## Open points

- Not tested yet. The first test target is Ubuntu 22.04 on SURF Research Cloud.
- `grycap.htcondor` lists Ubuntu 20.04 and 22.04 and EL. Ubuntu 24.04 is untested.
- SLURM on EL takes `slurm-drmaa` from the Galaxy depot. Only Ubuntu has been tested.
- HTCondor uses a shared pool password. Tokens would be better for a multi-site setup.
- Pulsar and relay client versions follow the ones usegalaxy.eu runs against the relay.
- The connection check covers the relay only. There is no check for RabbitMQ yet.
- The summary field names follow the BYOC form on Mirela Minkova's `byoc-variant2-relay` branch of the usegalaxy.eu playbook. They change if that form changes.
