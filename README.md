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

A host can be in both the `pulsar` and `workers` groups. That gives a single-VM endpoint for testing.

## Usage

```bash
ansible-galaxy install -r requirements.yml
cp inventory.example.ini inventory.ini        # add your hosts
cp secrets.example.yml secrets.yml            # add the queue URL and passwords
ansible-playbook site.yml -e @secrets.yml -e pulsar_scheduler=htcondor -e pulsar_connection=relay
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
- SLURM on EL needs a `slurm-drmaa` package source. Only Ubuntu is covered now.
- HTCondor uses a shared pool password. Tokens would be better for a multi-site setup.
- Pulsar and relay client versions follow the ones usegalaxy.eu runs against the relay.
