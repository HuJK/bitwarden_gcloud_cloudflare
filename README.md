# Bitwarden self-hosted on Google Cloud for Free

Bitwarden installation optimized for Google Cloud's 'always free' f1-micro compute instance

> _Note: if you follow these instructions the end product is a self-hosted instance of Bitwarden running in the cloud and will be free **unless** you exceed the 1GB egress per month or have egress to China or Australia. I talk about best practices to help avoid China/AUS egress, but there's a chance you can get charges from that so please keep that in mind._

This is a quick-start guide. Read about this project in more detail [here](https://bradford.la/2020/self-host-bitwarden-on-google-cloud).

---

## Features

* Bitwarden self-hosted
* Automatic backup databases to your github private repo.
* Automatic https certificate management through cloudflare proxy
* With Cloudflare Argo Tunnel, **You don't need to expose ANY port al all**.
* So, DDNS no longer required.

## Pre-requisites

Before you start, ensure you have the following:

1. A Google Cloud account
2. A Cloudflare-managed DNS record ready for Bitwarden.
3. A github account

## Step 1: Set up Google Cloud `e2-micro` Compute Engine Instance

Google Cloud offers an '[always free](https://cloud.google.com/free/)' tier of their Compute Engine with one virtual core and ~1000 MB of RAM (about 150 MB free depending on which OS you installed). [Bitwarden RS](https://github.com/dani-garcia/bitwarden_rs) runs well under these constraints; it's written in Rust and an ideal candidate for a micro instance. 

Go to [Google Compute Engine](https://cloud.google.com/compute) and open a Cloud Shell. You may also create the instance manually following [the constraints of the free tier](https://cloud.google.com/free/docs/gcp-free-tier). In the Cloud Shell enter the following command to build the properly spec'd machine: 

```bash
# create vm
$ gcloud compute instances create bitwarden-rs \
    --machine-type e2-micro \
    --zone us-west1-b \
    --image-project cos-cloud \
    --image-family cos-stable \
    --boot-disk-size=30GB \
    --scopes compute-rw
```

You may change the zone to be closer to you or customize the name (`bitwarden`), but most of the other values should remain the same. 

## Step 2: Pull and Configure Project

Enter a shell on the new instance and clone this repo:

```bash
$ git clone https://github.com/HuJK/bitwarden_gcloud_cloudflare
$ cd bitwarden_gcloud_cloudflare
```

Set up the docker-compose alias by using the included script:

```bash
$ sh utilities/install-alias.sh
$ source ~/.bashrc
$ docker-compose --version
docker-compose version 1.25.5, build 8a1c60f
```

### Step 2.1: Configure Environmental Variables with `.env`

I provide `.env.template` which should be copied to `.env` and filled out; filling it out is self-explanitory and requires certain values such as a domain name, github keys, etc. 

### Step 2.2: Setup backups

1. Create a empty private repo at your github account.
2. Generate a new SSH key pair and fill it to `.env` file
2. Setup your SSH key to the deploy keys to your repo, write access is required.

### Step 2.3: Setup Cloudflare argo tunnel

1. Download and install Cloudflare argo tunnrl from here: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation
2. Run `cloudflared tunnel login` locally and select your domain to get the cert.
3. Run `cloudflared tunnel create bwd`
4. Run `cloudflared tunnel route dns bwd bwd.example.com`
5. Copy `~/.cloudflared/XXXXXXX.json` to the cloudflared folder in this repo and rename it to tunnel.json.
6. Fill `CLOUDFLARED_TUNNEL_NAME` part in the env file
7. Make sure your desired domain for bitwarden are **not** exist in your Cloudflare DNS panel.  
   Cloudflared will create it later. If it exists, it fails.

## Configure Automatic Rebooting After Updates (_optional_)

Container-Optimized OS will automatically update itself, but the update will only be applied after a reboot. In order to ensure that you are using the most current operating system software, you can set a boot script that waits until an update has been applied to schedule a reboot.

Before you start, ensure you have `compute-rw` scope for your bitwarden compute vm. If you used the `gcloud` command above, it includes that scope. If not, go to your Google Cloud console and edit the "Cloud API access scopes" to have "Compute Engine" show "Read Write". You need to shut down your compute vm in order to change this.

Modify the script to set your local timezone and the time to schedule reboots: set the `TZ=` and `TIME=` variables in `utilities/reboot-on-update.sh`. By default the script will schedule reboots for 06:00 UTC. 

From within your compute vm console, type the command `toolbox`. From within `toolbox`, find the `utilities` folder within `bitwarden_gcloud`. `toolbox` mounts the host filesystem under `/media/root`, so go there to find the folder. It will likely be in `/media/root/home/<google account name>/bitwarden_gcloud/utilities` - `cd` to that folder.

Next, use `gcloud` to add the `reboot-on-update.sh` script to your vm's boot script metadata with the `add-metadata` [command](https://cloud.google.com/compute/docs/startupscript#startupscriptrunninginstances):

```bash
gcloud compute instances add-metadata bitwarden-rs --zone=us-west1-b --metadata-from-file startup-script=reboot-on-update.sh
```

You can confirm that your startup script has been added in your instance details under "Custom metadata" on the Compute Engine Console. 

Next, restart your vm with the command `$ sudo reboot`. Once your vm has rebooted, you can confirm that the startup script was run with the command:

```bash
$ sudo journalctl -u google-startup-scripts.service
```

Now the script will wait until a reboot is pending and then schedule a reboot for the time configured in the script.

## Configure Push Notifications(_optional_)
From `1.29.0`,Vaultwarden support automatically sync in mobile app depends on push notifications.

1. Go to https://bitwarden.com/host/ insert your email address and you'll get an INSTALLATION ID and KEY.

2. Insert the correct ID and the KEY from the previous step into `.env` file:

```yaml
PUSH_ENABLED=true
PUSH_INSTALLATION_ID=
PUSH_INSTALLATION_KEY=
```

## Step 3: Start Services

To start up, use `docker-compose`:

```bash
$ docker-compose up
```

You can now use your browser to visit your new Bitwarden site. 
