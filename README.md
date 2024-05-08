# Este Arch Repo

This repo hosts a private packages repository for my Arch Linux machines.

## How to replicate this?

### GPG key

We need a GPG key to sign our packages. I used a RSA key, with a 3072 keysize, valid for 5y.

```bash
gpg --full-generate-key 
```

Then, we can send the key to a key server:

```bash
gpg --send-keys 95A379AD219E338FE883D43027879E7CB656E9EE
```

And receive it:

```bash
gpg --recv-keys 95A379AD219E338FE883D43027879E7CB656E9EE
```

We then need to configure pacman and makepkg with this key:

```bash
sudo pacman-key --recv-keys 95A379AD219E338FE883D43027879E7CB656E9EE
sudo pacman-key --finger 95A379AD219E338FE883D43027879E7CB656E9EE  # verify the fingerprint
sudo pacman-key --lsign-key 95A379AD219E338FE883D43027879E7CB656E9EE  # locally sign

To configure makepkg, we just need to edit `/etc/makepkg.conf`:

```bash
#-- Packager: name/email of the person or organization building packages
PACKAGER="Esteban Thilliez <esteban.thilliez@gmail.com>"
#-- Specify a key to use for package signing
GPGKEY="27879E7CB656E9EE"
```

*Note: I used the short key in `GPGKEY` (last 16 characters of the GPG key)*

### Building signed packages

```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -sr --sign
```

We get a `*.pkg.tar.zst` and a `*.pkg.tar.zst.sign` file that we will add to our repo.


### Creating the repo

Basically, a package repository is just a directory available on the network. I like to use `git` for convenience.

```bash
mkdir repo
cd repo
git init
mkdir x86_64
cd x86_64
```

### Adding packages

To add a package, just copy the `.zst` and `.zst.sig` files and use `repo-add`:
```bash
mv ~/yay/*.zst* ./
repo-add --verify --sign reponame.db.tar.gz *.pkg.tar.zst
```

### Removing packages

```bash
repo-remove --verify --sign reponame.db.tar.gz yay
```

### Deploying the repo

This step may change depending on your setup. For me, I host the repo on Gitea and use `git-lfs` for `.zst` files (because they may be large). LFS files are stored on my NAS. Then, I deploy a Nginx webserver to serve my repo by rewriting the URLs to use those from Gitea. It's convenient, but it makes my repo dependent on my Gitea instance.

I run Nginx in Docker, and use Traefik as a reverse proxy. Everything could be done with Traefik but I prefer keeping things separate.

```yaml
# docker-compose.yml
---
services:
  web:
    image: nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    container_name: este_arch_repo-webserver
    ports:
      - 83:80
    restart: unless-stopped
    labels:
      - traefik.enable=true
      - traefik.http.routers.repo.entrypoints=websecure
      - traefik.http.routers.repo.rule=Host(`repo.example.com`)
      - traefik.http.routers.repo.tls=true
      - traefik.http.routers.repo.tls.certresolver=porkbun
```

```conf
# nginx.conf
http {

        server {

            location / {
                rewrite ^/(.*)$ https://gitea.example.com/estebanthi/este_arch_repo/media/branch/gitea/$1 permanent;
            }

            location = /x86_64/este.db {
                return 301 https://gitea.example.com/estebanthi/este_arch_repo/media/branch/gitea/x86_64/este.db.tar.gz;
            }

            location = /x86_64/este.db.sig {
                return 301 https://gitea.example.com/estebanthi/este_arch_repo/media/branch/gitea/x86_64/este.db.tar.gz.sig;
            }

            location = /x86_64/este.files {
                return 301 https://gitea.example.com/estebanthi/este_arch_repo/media/branch/gitea/x86_64/este.files.tar.gz;
            }

            location = /x86_64/este.files.sig {
                return 301 https://gitea.example.com/estebanthi/este_arch_repo/media/branch/gitea/x86_64/este.files.tar.gz.sig;
            }
        }

}

events {
  worker_connections  4096;
}
```

## Adding the repo to pacman

Add the following lines at the end of `/etc/pacman.conf`:

```conf
[reponame]
Server = https://repo.example.com/x86_64
```

Testing:

```bash
sudo pacman -Sy
sudo pacman -S yay
```
