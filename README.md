Awesome—here’s a **GitHub-ready, Amazon Linux–specific Chef pack** (Apache, NGINX, MariaDB, Docker) with correct package/service names and a simple way to run each cookbook locally on an EC2 instance.

---

# 📦 Repo layout

```
chef-amzn-pack/
├─ Policyfile.rb
├─ README.md
└─ cookbooks/
   ├─ al_apache/
   │  ├─ metadata.rb
   │  ├─ recipes/default.rb
   │  └─ templates/default/index.html.erb
   ├─ al_nginx/
   │  ├─ metadata.rb
   │  ├─ recipes/default.rb
   │  └─ templates/default/index.html.erb
   ├─ al_mariadb/
   │  ├─ metadata.rb
   │  └─ recipes/default.rb
   └─ al_docker/
      ├─ metadata.rb
      └─ recipes/default.rb
```

---

# 🧠 Policyfile (single entry point)

> Lets you run any cookbook via `-o recipe[...]` without needing a Chef Server.

```ruby
# Policyfile.rb
name 'chef-amzn-pack'

# Where to find external cookbooks (not used here; everything is local)
default_source :chef_repo, 'cookbooks'

# Default run_list (can be anything; we’ll override on the CLI anyway)
run_list 'al_apache::default'

cookbook 'al_apache',  path: 'cookbooks/al_apache'
cookbook 'al_nginx',   path: 'cookbooks/al_nginx'
cookbook 'al_mariadb', path: 'cookbooks/al_mariadb'
cookbook 'al_docker',  path: 'cookbooks/al_docker'
```

---

# 🍽️ Cookbook: `al_apache`

## `metadata.rb`

```ruby
name             'al_apache'
maintainer       'You'
license          'MIT'
description      'Apache (httpd) for Amazon Linux'
version          '0.1.0'
supports         'amazon'
```

## `recipes/default.rb`

```ruby
# al_apache::default
# Works on Amazon Linux 2 and Amazon Linux 2023 (httpd + systemd)

package 'httpd' do
  action :install
end

service 'httpd' do
  action [:enable, :start]
end

# Simple homepage
template '/var/www/html/index.html' do
  source 'index.html.erb'
  variables(
    hostname: node['hostname'],
    platform: node['platform'],
    platform_version: node['platform_version']
  )
  mode '0644'
end

# Basic firewall openness hint (security groups should handle 80/tcp on EC2)
# If you're using firewalld (rare on AL), uncomment:
# package 'firewalld' do
#   action :install
# end
# service 'firewalld' do
#   action [:enable, :start]
# end
# execute 'allow_http' do
#   command 'firewall-cmd --permanent --add-service=http && firewall-cmd --reload'
#   only_if "systemctl is-active --quiet firewalld"
# end
```

## `templates/default/index.html.erb`

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>Apache on Amazon Linux</title></head>
<body style="font-family: Arial, sans-serif;">
  <h1>✅ Hello from Apache (httpd) via Chef!</h1>
  <p><b>Host:</b> <%= @hostname %></p>
  <p><b>Platform:</b> <%= @platform %> <%= @platform_version %></p>
</body>
</html>
```

---

# 🌐 Cookbook: `al_nginx`

## `metadata.rb`

```ruby
name             'al_nginx'
maintainer       'You'
license          'MIT'
description      'NGINX for Amazon Linux'
version          '0.1.0'
supports         'amazon'
```

## `recipes/default.rb`

```ruby
# al_nginx::default

package 'nginx' do
  action :install
end

service 'nginx' do
  action [:enable, :start]
end

template '/usr/share/nginx/html/index.html' do
  source 'index.html.erb'
  variables(
    hostname: node['hostname']
  )
  mode '0644'
end
```

## `templates/default/index.html.erb`

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"><title>NGINX on Amazon Linux</title></head>
<body style="font-family: Arial, sans-serif;">
  <h1>🚀 NGINX is running (Chef-managed)</h1>
  <p>Host: <%= @hostname %></p>
</body>
</html>
```

---

# 🗄️ Cookbook: `al_mariadb`

> Amazon Linux commonly provides **MariaDB**. Service name: `mariadb`.

## `metadata.rb`

```ruby
name             'al_mariadb'
maintainer       'You'
license          'MIT'
description      'MariaDB server for Amazon Linux'
version          '0.1.0'
supports         'amazon'
```

## `recipes/default.rb`

```ruby
# al_mariadb::default
# Minimal MariaDB install + service start.
# (For production, use secure install steps / users / db grants.)

package %w(mariadb-server) do
  action :install
end

service 'mariadb' do
  action [:enable, :start]
end

# Optional: initialize root password ONLY if it's blank (basic demo).
# NOTE: This is simplistic; in production, use mysql_secure_installation or the mysql cookbook.
execute 'set-root-pass-if-empty' do
  command "mysqladmin -u root password 'rootpass'"
  only_if "mysql -u root -e 'show databases;'"
  returns [0] # allow success; if fails due to password existing, that's fine
  ignore_failure true
end

# Simple health check
execute 'mysql-ping' do
  command "mysql -u root -prootpass -e 'SELECT 1;'"
  returns [0,1] # allow failure if root pass not set
  ignore_failure true
end
```

---

# 🐳 Cookbook: `al_docker`

> Installs Docker engine from Amazon Linux repos and runs an NGINX container.

## `metadata.rb`

```ruby
name             'al_docker'
maintainer       'You'
license          'MIT'
description      'Docker engine + sample NGINX container on Amazon Linux'
version          '0.1.0'
supports         'amazon'
```

## `recipes/default.rb`

```ruby
# al_docker::default

# On AL2/AL2023, the package is 'docker'
package 'docker' do
  action :install
end

service 'docker' do
  action [:enable, :start]
end

# Allow current user to run docker without sudo (effective after re-login)
execute 'add-ec2-user-to-docker' do
  command 'usermod -aG docker ec2-user'
  only_if { Etc.getpwnam('ec2-user') rescue false }
end

# Pull and run NGINX container
execute 'docker-pull-nginx' do
  command 'docker pull nginx:latest'
  not_if "docker image ls | awk '{print $1\":\"$2}' | grep -q '^nginx:latest$'"
end

execute 'docker-run-nginx' do
  command 'docker run -d --name hello-nginx -p 8080:80 nginx:latest'
  not_if "docker ps --format '{{.Names}}' | grep -q '^hello-nginx$'"
end
```

---

# ▶️ How to run on an Amazon Linux EC2

> Works on **Amazon Linux 2** and **Amazon Linux 2023** t2.micro/t3.micro, 1 vCPU, 1–2 GB RAM is fine for these demos.

1. **Launch EC2** (Amazon Linux 2 or 2023) with **Security Group** allowing:

   * TCP **80** (for Apache/NGINX)
   * TCP **8080** (for Docker NGINX demo)
   * TCP **22** (SSH)

2. **SSH in** and install **Chef Workstation**
   *(Pick the right RPM URL from Chef’s site if versions change—this one is an example.)*

   ```bash
   sudo yum -y update || sudo dnf -y update
   # AL2 sometimes needs this for Chef binaries:
   sudo yum -y install libxcrypt-compat || true

   # Example (adjust version if needed)
   curl -LO https://packages.chef.io/files/stable/chef-workstation/24.4.1064/el/8/chef-workstation-24.4.1064-1.el8.x86_64.rpm
   sudo rpm -Uvh chef-workstation-*.rpm

   chef --version
   ```

3. **Get this repo onto the instance**

   ```bash
   # If you uploaded to GitHub:
   git clone https://github.com/<your-org>/chef-amzn-pack.git
   cd chef-amzn-pack
   ```

   Or copy/paste the structure/files above into the same paths.

4. **Vendor the policy (generates lock)**

   ```bash
   chef install
   ```

5. **Run any cookbook in local (zero) mode**

   * **Apache (httpd)**

     ```bash
     sudo chef-client -z -o 'recipe[al_apache]'
     # Test: curl http://localhost/  (or open your EC2 public IP in a browser)
     ```

   * **NGINX**

     ```bash
     sudo chef-client -z -o 'recipe[al_nginx]'
     # Test: curl http://localhost/  (or open EC2 public IP)
     ```

   * **MariaDB**

     ```bash
     sudo chef-client -z -o 'recipe[al_mariadb]'
     # Optional test:
     mysql -u root -prootpass -e 'SHOW DATABASES;' || echo "Root pass not set (ok for demo)"
     ```

   * **Docker (with demo container)**

     ```bash
     sudo chef-client -z -o 'recipe[al_docker]'
     # Test: curl http://localhost:8080/  (or open EC2 public IP:8080)
     # If you want docker without sudo immediately:
     newgrp docker
     docker ps
     ```

---

# 🧪 Quick troubleshooting

* **Package not found** on AL2023 → run `sudo dnf makecache` then retry.
* **Port not reachable** → open the Security Group for 80/8080 inbound from your IP or 0.0.0.0/0 (for demo).
* **Docker permission denied** → re-login or `newgrp docker` after running the Docker recipe.
* **Chef Workstation deps on AL2** → `sudo yum install -y libxcrypt-compat`.

---

# 📝 README.md (drop this in the repo root)

````markdown
# Chef Amazon Linux Pack

Cookbooks tailored for Amazon Linux 2/2023:
- `al_apache` — Apache httpd
- `al_nginx` — NGINX
- `al_mariadb` — MariaDB server
- `al_docker` — Docker engine + demo NGINX container

## Quickstart

```bash
sudo yum -y update || sudo dnf -y update
sudo yum -y install libxcrypt-compat || true
curl -LO https://packages.chef.io/files/stable/chef-workstation/24.4.1064/el/8/chef-workstation-24.4.1064-1.el8.x86_64.rpm
sudo rpm -Uvh chef-workstation-*.rpm
git clone https://github.com/<your-org>/chef-amzn-pack.git
cd chef-amzn-pack
chef install
sudo chef-client -z -o 'recipe[al_apache]'
````

## Run others

```bash
sudo chef-client -z -o 'recipe[al_nginx]'
sudo chef-client -z -o 'recipe[al_mariadb]'
sudo chef-client -z -o 'recipe[al_docker]'
```

## Test

* Apache/NGINX: `curl http://localhost/` or open `http://<EC2-PUBLIC-IP>/`
* Docker demo: `curl http://localhost:8080/` or `http://<EC2-PUBLIC-IP>:8080/`

```

---

If you’d like, I can bundle this into a **zip** or push to a **new GitHub repo name** you prefer (e.g., `chef-amzn-pack`).
```
