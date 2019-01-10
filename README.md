# Playing with GitLab

## Installing GitLab

- [Install GitLab using Docker Compose](https://docs.gitlab.com/omnibus/docker/README.html#install-gitlab-using-docker-compose)

**Update `docker-compose.yml`**

- Setup host name. Make sure that `/etc/hosts` is updated if you use localhost for hosting the service.
- Change port numbers if necessary.
- Change local folder paths in `volumes` section.

### Notes

- Don't use port `9090`, it is reserved for Prometheus
- Admin area: `{url}/admin`
- Default admin user: `root`

## Install CI component: GitLab Runner

- [Install GitLab Runner with Docker](https://docs.gitlab.com/runner/install/docker.html)
- Docker image is added to the `docker-compose.yml`.
- Setup configuration. [Instruction](https://docs.gitlab.com/runner/register/index.html#docker)

The following command launch the setup script. Run it in your project folder. The `docker-compose.yml` map the `./gitlab-runner` folder to the runner container.

```bash
$ docker run --rm -t -i -v $(pwd)/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner:alpine register
```

Make sure that always register a new runner with the above command. Don't edit token manually in `config.toml`.

Alternative option. In this case you have to install `gitlab-runner` in your host OS.

```
brew install gitlab-runner
gilab-runner register
```

It will create the config `toml` in `~/.gitlab-runner/config.toml

Setup config volume in `docker-compose.yml`.

Important! Update `volumes` to `volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]` in `config.toml`.

Don't forget to copy certs in the `certs` folder also.

## Configuring Traefik (failed: ssh not supported)

- https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-ubuntu-16-04

- Problem with Traefik that it is not supporting ssh

## Setup dnsmasq with loopback alias

Articles:

- [Using dnsmasq and loopback alias to solve Docker for Mac networking issue](https://medium.com/@williamhayes/local-dev-on-docker-fun-with-dns-85ca7d701f0a)
- [Use dnsmasq instead of etc hosts](https://www.stevenrombauts.be/2018/01/use-dnsmasq-instead-of-etc-hosts/)
- [Persistent loopback on macOS](https://blog.felipe-alfaro.com/2017/03/22/persistent-loopback-interfaces-in-mac-os-x/)

1. Setup loopback alias

Temporarly loopback alias

```bash
sudo ifconfig lo0 alias 10.254.254.254
```

Permanent loopback alias

```
$ cat << EOF | sudo tee -a /Library/LaunchDaemons/my.awesome.loopback.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>my.awesome.loopback</string>
    <key>ProgramArguments</key>
    <array>
        <string>/sbin/ifconfig</string>
        <string>lo0</string>
        <string>alias</string>
        <string>10.254.254.254</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
  </dict>
</plist>
EOF
```

2. Setup dnsmasq

Install `dnsmasq` with brew. `dnsmasq` website: http://www.thekelleys.org.uk/dnsmasq/doc.html

```
brew install dnsmasq
```

Add your own personal config file to the default config file:

```
# Create an `/etc` folder in your brew installation folder if not exists
$ mkdir $(brew --prefix)/etc
$ cat << EOF | tee $(brew --prefix)/etc/dnsmasq.conf
conf-file=/Users/yourname/.dnsmasq/dnsmasq.conf
EOF
```

Create your personal config file in `~/.dnsmasq/dnsmasq.conf`

(Don't use `.dev`, Chrome automatically redirect it to `https://something.dev`.)

```
cat << EOF | tee ~/.dnsmasq/dnsmasq.conf
server=8.8.8.8
server=8.8.4.4
address=/.local/10.254.254.254
address=/.test/10.254.254.254
address=/.loc/10.254.254.254
EOF
```

(You can use `tee -a` option to append content to the file instead of overwriting it.)

The full sample config file:  https://github.com/imp/dnsmasq/blob/master/dnsmasq.conf.example

Restart dnsmasq

```
sudo brew services restart dnsmasq
```

## Setup Registry

- Add configuration to the docker-compose.yml
- Binding ports does not need.

Generate self signed cert: https://www.digitalocean.com/community/tutorials/how-to-create-an-ssl-certificate-on-nginx-for-ubuntu-14-04

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./ssl/gitlab.docker.loc.key -out ./ssl/gitlab.docker.loc.crt
```

For the registry we need proper ssl certificate, but only if we use public domain.
- [Let's Encrypt](https://letsencrypt.org/getting-started/)

We can use `certbot`. https://certbot.eff.org/

```
brew install certbot
```

For localhost development, we can add our previously generated `.crt` to Keychain.

Copy to `~/.docker/certs.d/gitlab.docker.loc:5050/client.crt` and `client.key`

Setup `certs` folder for gitlab-runner. [doc](https://github.com/ayufan/gitlab-ci-multi-runner/blob/master/docs/install/docker.md#installing-trusted-ssl-server-certificates)
