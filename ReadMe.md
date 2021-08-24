
<p align="center">
  <img width="1000" height="350" src="https://res.cloudinary.com/practicaldev/image/fetch/s--teMhd-TQ--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://user-images.githubusercontent.com/567298/71035636-8d37b900-2124-11ea-96e2-4bb9fd4c8194.png">
</p>


# What is Concourse CI:
Concourse CI is a Continuous Integration Platform. Concourse enables you to construct pipelines with yaml configuration that can consist out of 3 core concepts that compose them:
- tasks
- resources
- jobs

For more information about this have a look at their [doc](https://concourse.ci/concepts.html)

## What will we be doing today
We will setup a Concourse Server on Ubuntu 18.04 and run the traditional Hello, World pipeline

## Setup the Server:
Concourse requires PostgresSQL for it's database (at least version 9.3)

```
apt update && apt upgrade -y
apt install postgresql postgresql-contrib -y
systemctl enable postgresql
```
Create the Database and User for Concourse on your Postgres database:
```
sudo -u postgres createuser concourse
sudo -u postgres createdb --owner=concourse atc
```
Download the Concourse Server Archive:
```
export CONCOURSE_VERSION=5.7.2
wget https://github.com/concourse/concourse/releases/download/v${CONCOURSE_VERSION}/concourse-${CONCOURSE_VERSION}-linux-amd64.tgz
```
Extract the concourse archive:
```
tar -zxf concourse-${CONCOURSE_VERSION}-linux-amd64.tgz -C /usr/local
```
Create the encryption keys:
```
mkdir /etc/concourse
ssh-keygen -t rsa -q -N '' -f /etc/concourse/tsa_host_key
ssh-keygen -t rsa -q -N '' -f /etc/concourse/worker_key
ssh-keygen -t rsa -q -N '' -f /etc/concourse/session_signing_key
cp /etc/concourse/worker_key.pub /etc/concourse/authorized_worker_keys
```

## Concourse Web Process Configuration:
```
EXT_IP=$(curl -s https://ip.ruan.dev)

cat > /etc/concourse/web_environment << EOF
CONCOURSE_ADD_LOCAL_USER=admin:admin
CONCOURSE_SESSION_SIGNING_KEY=/etc/concourse/session_signing_key
CONCOURSE_TSA_HOST_KEY=/etc/concourse/tsa_host_key
CONCOURSE_TSA_AUTHORIZED_KEYS=/etc/concourse/authorized_worker_keys
CONCOURSE_POSTGRES_HOST=127.0.0.1
CONCOURSE_POSTGRES_USER=concourse
CONCOURSE_POSTGRES_PASSWORD=concourse
CONCOURSE_POSTGRES_DATABASE=atc
CONCOURSE_MAIN_TEAM_LOCAL_USER=admin
CONCOURSE_EXTERNAL_URL=http://${EXT_IP}:8080
EOF
```

Concourse Worker Process Configuration:

```
cat > /etc/concourse/worker_environment << EOF
CONCOURSE_WORK_DIR=/var/lib/concourse
CONCOURSE_TSA_HOST=127.0.0.1:2222
CONCOURSE_TSA_PUBLIC_KEY=/etc/concourse/tsa_host_key.pub
CONCOURSE_TSA_WORKER_PRIVATE_KEY=/etc/concourse/worker_key
EOF
```

Create a Concourse user:

```
mkdir /var/lib/concourse
sudo adduser --system --group concourse
sudo chown -R concourse:concourse /etc/concourse /var/lib/concourse
sudo chmod 600 /etc/concourse/*_environment
```

Create SystemD Unit Files, first for the Web Service:

```
cat > /etc/systemd/system/concourse-web.service << EOF
[Unit]
Description=Concourse CI web process (ATC and TSA)
After=postgresql.service

[Service]
User=concourse
Restart=on-failure
EnvironmentFile=/etc/concourse/web_environment
ExecStart=/usr/local/concourse/bin/concourse web

[Install]
WantedBy=multi-user.target
EOF
```
Then the SystemD Unit File for the Worker Service:

```
cat  > /etc/systemd/system/concourse-worker.service << EOF
[Unit]
Description=Concourse CI worker process
After=concourse-web.service

[Service]
User=root
Restart=on-failure
EnvironmentFile=/etc/concourse/worker_environment
ExecStart=/usr/local/concourse/bin/concourse worker

[Install]
WantedBy=multi-user.target
EOF
```

Create a postgres password for the concourse user:
```
cd /home/concourse/

sudo -u concourse psql atc
atc=> ALTER USER concourse WITH PASSWORD 'concourse';
atc=> \q
```

Start and Enable the Services:

```
systemctl start concourse-web concourse-worker
systemctl enable concourse-web concourse-worker postgresql
systemctl status concourse-web concourse-worker

systemctl is-active concourse-worker concourse-web
active
active
```

The listening ports should more or less look like the following:


```

**netstat -tulpn**

Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN      11290/postgres
tcp        0      0 127.0.0.1:7776          0.0.0.0:*               LISTEN      13040/concourse
tcp        0      0 127.0.0.1:7777          0.0.0.0:*               LISTEN      13118/gdn
tcp        0      0 127.0.0.1:7787          0.0.0.0:*               LISTEN      13040/concourse
tcp        0      0 127.0.0.1:7788          0.0.0.0:*               LISTEN      13040/concourse
tcp        0      0 127.0.0.1:8079          0.0.0.0:*               LISTEN      13039/concourse
tcp6       0      0 :::41299                :::*                    LISTEN      13039/concourse
tcp6       0      0 :::8888                 :::*                    LISTEN      13040/concourse
tcp6       0      0 ::1:5432                :::*                    LISTEN      11290/postgres
tcp6       0      0 :::35323                :::*                    LISTEN      13039/concourse
tcp6       0      0 :::2222                 :::*                    LISTEN      13039/concourse
tcp6       0      0 :::8080                 :::*                    LISTEN      13039/concourse
udp        0      0 0.0.0.0:54582           0.0.0.0:*                           13118/gdn

```
Client Side:
When you access the UI on the external ip on port 8080:

<p align="center">
  <img width="1000" height="250" src="https://res.cloudinary.com/practicaldev/image/fetch/s--mD4GPV0d--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://user-images.githubusercontent.com/567298/71032496-244d4280-211e-11ea-8f28-5b5a0fff7ae0.png">
</p>

After logging in, you should see a screen with no pipelines configured:

<p align="center">
  <img width="1000" height="450" src="https://res.cloudinary.com/practicaldev/image/fetch/s--eeQ_ldHW--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://user-images.githubusercontent.com/567298/71032574-4941b580-211e-11ea-958c-c047ac718af7.png">
</p>

We require the fly cli client which is used to interact with concourse. On your local machine, download the fly archive, extract the archive and move the fly binary to your path.

I will by using the client for Mac, for alternate versions have a look at their [https://github.com/concourse/concourse/releases](releases) page.


```
wget https://github.com/concourse/concourse/releases/download/v3.10.0/fly_linux_amd64 -O /usr/bin/fly

chmod +x /usr/bin/fly
```

Now you have to make `~/.flyrc` file using nano/vi/vim
`vim ~/.flyrc`

```
targets:
  ci:
    api: http://public_ip:8080
    team: main
```


Follow the URL and your target should be saved. Lets list our targets:

`fly targets`

Listing Registered Workers:
`fly -t ci workers`
this will show you no targets and refer a command `fly sync ci` to download the updated fly version.
Next, we need to setup our Concourse Target by Authenticating against our Concourse Endpoint, lets setup our target with the name ci

```
fly -t ci login -c http://${EXT_IP}:8080
logging in to team 'main'

navigate to the following URL in your browser:

  http://public_ip/login?fly_port=49722

or enter token manually:
```

enter the bearer token in `~/.flyrc` file just below the `team section`.

```
targets:
  tutorial:
    api: http://127.0.0.1:8080
    team: main
    token:
      type: Bearer
      value: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJjc3JmIjoiYjE3ZDgxZmMwMWIxNDE1Mjk2OWIyZDc4NWViZmVjM2EzM2IyY2MxYWZjZjU3Njc1ZWYwYzY0MTM3MWMzNzI3OSIsImV4cCI6MTUyMjcwMjUwMCwiaXNBZG1pbiI6dHJ1ZSwidGVhbU5hbWUiOiJtYWluIn0.JNutBGQJMKyFzow5eQOTXAw3tOeM8wmDGMtZ-GCsAVoB7D1WHv-nHIb3Rf1zWw166FuCrFqyLYnMroTlQHyPQUTJFDTiMEGnc5AY8wjPjgpwjsjyJ465ZX-70v1J4CWcTHjRGrB1XCfSs652s8GJQlDf0x2hi5K0xxvAxsb0svv6MRs8aw1ZPumguFOUmj-rBlum5k8vnV-2SW6LjYJAnRwoj8VmcGLfFJ5PXGHeunSlMdMNBgHEQgmMKf7bFBPKtRuEAglZWBSw9ryBopej7Sr3VHPZEck37CPLDfwqfKErXy_KhBA_ntmZ87H1v3fakyBSzxaTDjbpuOFZ9yDkGA
```

Listing Active Containers:
```
fly -t ci containers
handle  worker  pipeline  job  build #  build id  type  name  attempt
```

If you are running Concourse on Ubuntu 18 and you ran into this [https://github.com/concourse/concourse/issues/374](issue):
```
sudo apt install resolvconf
echo -e "nameserver 8.8.4.4\nnameserver 8.8.8.8" > /etc/resolvconf/resolv.conf.d/head
sudo service resolvconf restart
```


Now create a pipeline in .yml file. Here i am integrating terraform with concourse so, i am cloning terraform code from my github account and then running in the cocourse ci pipeline.

create a terraform.yml file and run the below commands.

# Set the pipeline

```
fly -t ci set-pipeline -p  terraform_aws -c terraform.yml
```

<p align="center">
  <img width="1000" height="525" src="https://github.com/amit17133129/Concourse/blob/main/images/aws_terraform_pipeline_creation.jpg?raw=true">
</p>
you will see the beow status in the dashboard

<p align="center">
  <img width="1000" height="525" src="https://github.com/amit17133129/Concourse/blob/main/images/pipeline_unpaused.jpg?raw=true">
</p>

# Unpause the pipeline
```
fly -t ci unpause-pipeline -p  terraform_aws
```
<p align="center">
  <img width="1000" height="80" src="https://github.com/amit17133129/Concourse/blob/main/images/unpause%20pipeline.jpg?raw=true">
</p>

you will see the beow status in the dashboard

<p align="center">
  <img width="1000" height="525" src="https://github.com/amit17133129/Concourse/blob/main/images/pipeline_pending_state.jpg?raw=true">
</p>

# Trigger the pipeline
```
fly -t ci trigger-job --job  terraform_aws/terraform
```
<p align="center">
  <img width="1000" height="80" src="https://github.com/amit17133129/Concourse/blob/main/images/pipeline%20started.jpg?raw=true">
</p>
<p align="center">
  <img width="1000" height="525" src="https://github.com/amit17133129/Concourse/blob/main/images/running%20status.jpg?raw=true">
</p>



<p align="center">
  <img width="1000" height="525" src="https://github.com/amit17133129/Concourse/blob/main/images/terraform%20pipeline_job.jpg?raw=true">
</p>


<p align="center">
  <img width="1000" height="525" src="https://github.com/amit17133129/Concourse/blob/main/images/pulled%20centos%20image.jpg?raw=true">
</p>

# Terraform init
<p align="center">
  <img width="1000" height="525" src="https://github.com/amit17133129/Concourse/blob/main/images/terraform%20init.jpg?raw=true">
</p>

## terraform Apply
<p align="center">
  <img width="1000" height="525" src="https://github.com/amit17133129/Concourse/blob/main/images/terraform%20apply.jpg?raw=true">
</p>
