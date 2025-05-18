> **Quick Start:** Jump to the [Start Here](#start-here) section for step-by-step setup instructions.

# Supabase on DigitalOcean

[Supabase](https://supabase.com/) is a backend-as-a-service platform built around the Postgres database, and is an Open Source alternative to Firebase. It can reduce time to market by providing a ready to use backend that includes a database with real time capabilities, authentication, object storage and edge functions. You can use Supabase as a service via their [managed offerings](https://supabase.com/pricing) or self-host it on your own server or on a cloud provider.

If you want to jump to the start of the project, you can do so by following the [Start](#start-here) section. This will guide you through the steps to get a self-hosted Supabase instance up and running on DigitalOcean.

## Running Supabase on DigitalOcean

We will self-host Supabase by deploying the following architecture.
![Supabase on DigitalOcean](./assets/Supabase-on-DO-white-bkg.png "Supabase on DigitalOcean")

### Docker Compose

The components that make up Supabase will be running via a [docker-compose.yml](./packer/supabase/docker-compose.yml) file. The following is taken directly from the Supabase [self-hosting documentation](https://supabase.com/docs/guides/self-hosting) page and provides a description of each of its components:

> - [Kong](https://github.com/Kong/kong) is a cloud-native API gateway.
> - [GoTrue](https://github.com/netlify/gotrue) is an SWT based API for managing users and issuing SWT tokens.
> - [PostgREST](http://postgrest.org/) is a web server that turns your PostgreSQL database directly into a RESTful API
> - [Realtime](https://github.com/supabase/realtime) is an Elixir server that allows you to listen to PostgreSQL inserts, updates, and deletes using websockets. Realtime pollsPostgres' built-in replication functionality for database changes, converts changes to JSON, then broadcasts the JSON over websockets to authorized clients.
> - [Storage](https://github.com/supabase/storage-api) provides a RESTful interface for managing Files stored in S3, using Postgres to manage permissions.
> - [postgres-meta](https://github.com/supabase/postgres-meta) is a RESTful API for managing your Postgres, allowing you to fetch tables, add roles, and run queries, etc.
> - [PostgreSQL](https://www.postgresql.org/) is an object-relational database system with over 30 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

In addition to the above components, the docker-compose file also runs [swag](https://docs.linuxserver.io/general/swag). SWAG (Secure Web Application Gateway) provides an Nginx webserver and reverse proxy with a built-in certbot client that automates free SSL certificate generation and renewal. It also contains [fail2ban](https://www.fail2ban.org/wiki/index.php/Main_Page) for added intrusion prevention. As swag deploys Nginx we will also use it to setup basic authentication to protect access to `studio` (the dashboard component of Supabase).

### DigitalOcean Components

All of the above will be running on a DigitalOcean [Droplet](https://www.digitalocean.com/products/droplets). Persistent storage for the database is provided via a [Volume](https://www.digitalocean.com/products/block-storage) attached to the Droplet and object storage, for artifacts like profile pics and more, will be achieved using [Spaces](https://www.digitalocean.com/products/spaces ). A Domain, Reserved IP and Firewall are also setup to ensure we can securely access our Supabase instance from the web.

### SendGrid

Supabase's auth component, `GoTrue`, requires the ability to send emails. As DigitalOcean blocks Port 25 on all Droplets for new accounts (IP reputation being a main reason for this as well as [other factors](https://www.digitalocean.com/community/tutorials/why-you-may-not-want-to-run-your-own-mail-server)) we will use [SendGrid](https://sendgrid.com/) to send emails. SendGrid offers a generous free plan of 100 emails/day which should suffice for most use cases.

### Packer and Terraform

At DigitalOcean [simplicity in all we DO](https://www.digitalocean.com/about) is one of our core values, and automating as much as possible of our processes enables us to achieve this. In this regard we will use [Packer](https://www.packer.io/) and [Terraform](https://www.terraform.io/) to automate the build and provision the resources.

# Start here

## Table of Contents
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Step-by-Step Guide](#step-by-step-guide)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Configure Packer](#2-configure-packer)
  - [3. Build the Droplet Snapshot](#3-build-the-droplet-snapshot)
  - [4. Configure Terraform](#4-configure-terraform)
  - [5. Deploy Infrastructure](#5-deploy-infrastructure)
  - [6. Retrieve Credentials](#6-retrieve-credentials)
- [Extra Information & Troubleshooting](#extra-information--troubleshooting)
- [Credits](#credits)

---

## Prerequisites
Before you begin, ensure you have the following:

- **DigitalOcean Account** ([Sign up](https://cloud.digitalocean.com/login))
- **SendGrid Account** ([Sign up](https://signup.sendgrid.com/))
- **Domain Name** (added to DigitalOcean DNS)
  - To add your domain, go to the [DigitalOcean Domains dashboard](https://cloud.digitalocean.com/networking/domains), click "Add Domain," and follow the instructions. Make sure to update your domain registrar's nameservers to point to DigitalOcean. See the [official guide](https://docs.digitalocean.com/products/networking/dns/how-to/add-domains/) for step-by-step help.
- **Packer CLI** ([Install guide](https://developer.hashicorp.com/packer/tutorials/docker-get-started/get-started-install-cli))
- **Terraform CLI** ([Install guide](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli))
- **curl** (usually pre-installed on Linux/macOS)

> **Have you completed all prerequisites above?**
> Only continue if you have created your accounts, set up your domain in DigitalOcean DNS, and installed the required tools.

---

## Quick Start
1. **Clone the repository**
2. **Configure Packer variables**
3. **Build the snapshot**
4. **Configure Terraform variables**
5. **Deploy with Terraform**
6. **Access your Supabase instance**

---

## Step-by-Step Guide

### 1. Clone the Repository
```sh
git clone https://github.com/digitalocean/supabase-on-do.git
cd supabase-on-do
```

### 2. Configure Packer
```sh
cd packer
cp supabase.auto.pkrvars.hcl.example supabase.auto.pkrvars.hcl
# Edit supabase.auto.pkrvars.hcl with your DigitalOcean API token and region
```

In `supabase.auto.pkrvars.hcl`, you must set at least the following variables:

```hcl
############
# IMP. The below token should not be stored in version control
############
do_token = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

## REQUIRED ##
region = "ams3"

## OPTIONAL ##
# droplet_image = ""
# droplet_size  = ""
# tags          = [""]
```

- Replace `do_token` with your DigitalOcean API token.
- Set `region` to your chosen region (e.g., `nyc3`, `sfo3`, etc.).
- Optionally, adjust `droplet_image`, `droplet_size`, and `tags` as needed.

- **Get your DigitalOcean API token:** [Create here](https://docs.digitalocean.com/reference/api/create-personal-access-token/)
- **Recommended region:** e.g., `nyc3`, `sfo3`, etc.
  - To see all available regions, you can install the [doctl CLI](https://docs.digitalocean.com/reference/doctl/how-to/install/) and run:
    ```sh
    doctl compute region list
    ```
  - For best performance, use the same region for both your Supabase instance and any apps that will connect to it. (Not required, but recommended.)

### 3. Build the Droplet Snapshot

> **Note:** If you get errors about the Packer version, open `packer/supabase.pkr.hcl` and update the `required_version` field to match your installed Packer version. For major version changes (e.g., 1 → 2), check the [Packer documentation](https://developer.hashicorp.com/packer/docs) for breaking changes.

```sh
packer init .
packer build .
```

### 4. Configure Terraform
```sh
cd ../terraform
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your variables (see comments in the file)
```

In `terraform.tfvars`, you must set at least the following variables:

```hcl
## REQUIRED SECRETS ##
############
# IMP. The below secrets/tokens/apis should not be stored in version control
############
do_token                 = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
sendgrid_api             = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
spaces_access_key_id     = "xxxxxxxxxxxxxxxxxxxx"
spaces_secret_access_key = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

## REQUIRED ##
region    = "ams3"
domain    = "example.com"
timezone  = "Europe/Amsterdam"
auth_user = "admin"
site_url  = "app.example.com"

smtp_admin_user = "admin@example.com"
smtp_addr       = "Company Address"
smtp_city       = "Company City"
smtp_country    = "Company Country"

## OPTIONAL ##
# droplet_image        = ""
# droplet_size         = ""
# droplet_backups      = false
# ssh_pub_file         = ""
# ssh_keys             = [12345678, 87654321]
# tags                 = ["admin", "tim"]
# volume_size          = 100
# enable_ssh           = true
# ssh_ip_range         = ["your-own-ip", "office-ip/24"]
# enable_db_con        = false
# db_ip_range          = ["your-own-ip", "office-ip/24"]
# spaces_restrict_ip   = true
# studio_org           = ""
# studio_project       = ""
# smtp_host            = ""
# smtp_port            = ""
# smtp_user            = ""
# smtp_sender_name     = ""
# smtp_addr_2          = ""
# smtp_state           = ""
# smtp_zip_code        = ""
# smtp_sender_reply_to = ""
# smtp_nickname        = ""
```

- Replace all secrets/tokens with your actual credentials.
- Set `region` to your chosen region (e.g., `nyc3`, `sfo3`, etc.).
- Set `domain` to your own domain (e.g., `example.com`).
- Set `site_url` to your app's URL. **Recommended:** use a `supabase.` prefix, e.g., `supabase.example.com`.
- Set `timezone` to your local timezone (e.g., `Europe/Amsterdam`). You can find your timezone in this format at [TimeZoneDB](https://timezonedb.com/time-zones) or [List of tz database time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).
- Set `auth_user` and SMTP fields as needed for your setup.
- In the OPTIONAL section, consider setting `ssh_ip_range` and `db_ip_range` to restrict SSH and database access to your own IPs for better security.

- **Get your SendGrid API key:** [Create here](https://docs.sendgrid.com/for-developers/sending-email/brite-verify#creating-a-new-api-key)
- **Add your domain to DigitalOcean DNS:** [Guide](https://docs.digitalocean.com/products/networking/dns/how-to/add-domains/)

### 5. Deploy Infrastructure
> **Note:** If you get errors about the Terraform version, open `terraform/provider.tf` and update the `required_version` field to match your installed Packer version. For major version changes (e.g., 1 → 2), check the [Terraform documentation](https://developer.hashicorp.com/terraform/docs) for breaking changes.

```sh
terraform init
terraform apply
# Confirm at the prompt (or use --auto-approve for automation)
```

> **Important:** You must run `terraform apply` a second time after the first apply completes. This is required because some resources (like SendGrid components) depend on DNS records that are only created during the first run. Running it twice ensures all resources are fully provisioned and configured correctly.

```sh
terraform apply
```

### 6. Retrieve Credentials

After running `terraform apply` (twice), you will need to retrieve the credentials generated for your Supabase instance:

**1. Get your credentials:**
```sh
terraform output htpasswd         # Basic auth password
terraform output psql_pass        # PostgreSQL password
terraform output jwt              # JWT secret
terraform output jwt_anon         # JWT anon token
terraform output jwt_service_role # JWT service role token
```

**2. Access your Supabase instance:**
- Open your browser and go to: `https://supabase.<your-domain>`
- When prompted for authentication:
  - **Username:** Use the `auth_user` value you set in step 4 (`terraform.tfvars`).
  - **Password:** Use the value from `terraform output htpasswd` above.

**3. Understanding the JWT outputs:**
- `jwt_anon` and `jwt_service_role`:
  - These are the keys you will use to connect external services (such as n8n, backend apps, etc.) to your Supabase instance.
  - Use `jwt_anon` for anonymous access and `jwt_service_role` for service-level access.
- `jwt`:
  - This is your Supabase JWT secret. **It is highly sensitive!**
  - Do not share this secret or store it in any insecure location. Keep it private and secure at all times.

---

## Extra Information & Troubleshooting

- **Why SendGrid?** DigitalOcean blocks port 25, so SendGrid is used for email delivery.
- **Docker Compose** is used to run all Supabase components, including Kong, GoTrue, PostgREST, Realtime, Storage, postgres-meta, PostgreSQL, and SWAG (for SSL and reverse proxy).
- **Packer** builds a custom Droplet image with all dependencies.
- **Terraform** provisions the Droplet, storage, networking, and configures DNS.
- **Domain Setup:** Make sure your domain's nameservers point to DigitalOcean.
- **Common Issues:**
  - API tokens missing or incorrect: Double-check your `.hcl` and `.tfvars` files.
  - DNS not propagating: Wait a few minutes or check your registrar's settings.
  - For more help, see the [official Supabase self-hosting docs](https://supabase.com/docs/guides/self-hosting) or [DigitalOcean documentation](https://docs.digitalocean.com/).

---

## Credits
- [Supabase](https://supabase.com/)
- [DigitalOcean](https://www.digitalocean.com/)
- [SendGrid](https://sendgrid.com/)
- [Packer](https://www.packer.io/)
- [Terraform](https://www.terraform.io/)

README rewrite by [J-Pster](https://github.com/J-Pster)

---

Enjoy your self-hosted Supabase instance on DigitalOcean!
