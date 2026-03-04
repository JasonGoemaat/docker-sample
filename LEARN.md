# Building Sveltekit and Docker Image

## Building Docker Image

From: https://www.youtube.com/watch?v=GLQ3PXDgbIk

Used standard steps to create new sveltekit demo project with typescript
and tailwindcss.

    pnpm create svelte@latest sample
    cd sample
    pnpm i

Added node adapter package:

    pnpm i -D @sveltejs/adapter-node

In [svelte.config.js](svelte.config.js):

    //import adapter from '@sveltejs/adapter-auto';
    import adapter from '@sveltejs/adapter-node';

Created [.dockerignore](.dockerignore) to ignore node_modules.

Created [dockerfile](dockerfile) with build step:

    FROM node:lts-alpine as build
    WORKDIR /app
    COPY ./package*.json ./
    RUN npm install
    COPY . .
    RUN npm run build

    FROM node:lts-alpine as production
    COPY --from=build /app/build .
    COPY --from=build /app/package.json .
    COPY --from=build /app/package-lock.json .
    RUN npm ci --omit dev
    EXPOSE 3000
    CMD ["node", "."]

Build docker image:

    docker build . -t sample-image

Test docker image:

    docker run -p 3000:3000 --name sample-container sample-image

Now that it's been run for the first time, it can be started later with:

    docker start sample-image

## Build and run from github container registry

Fedora contains podman which is pretty much just like docker.  I installed
this to enable the docker cli compatability layer so I can use 'docker'
isntead of 'podman' in the shell:

    sudo dnf install podman-docker

From: https://www.youtube.com/watch?v=f5AlQE0i5m0

Create PAT using [this link](https://www.youtube.com/redirect?event=video_description&redir_token=QUFFLUhqbkVUcmluRDNUZFR5R2xTcmM4cFNHbFM4YUtrd3xBQ3Jtc0tsUGFpRVhZaENSeC00WEdHUURNam1jYkF3YUIzMzZUd0MyY1ltcnlZTFFkUWVqWDc4TmN6VVVEMmJhdUdibXNoc1RKakc4ZmR3MkNOdUhrU1h3UGtOQ1Q2WVd2VE5VdE0tWkxqc1pNaXlpM2k2ZEtsdw&q=https%3A%2F%2Fgithub.com%2Fsettings%2Ftokens%2Fnew%3Fscopes%3Dwrite%3Apackages%2Cread%3Apackages%2Cdelete%3Apackages&v=f5AlQE0i5m0) -
use default permissions of just 'write:packages' (which includes 
'read:packages') and 'delete:packages'.   I store this in the file `.secret`
and added that to `.gitignore`.

Login to github container registry, pasting PAT to stdin:

    docker login ghcr.io -u JasonGoemaat --password-stdin

Build and upload image:

    docker build . -t ghcr.io/jasongoemaat/docker-image-github:latest
    docker push ghcr.io/jasongoemaat/docker-image-github:latest

Then on the server, export PAT to environment variable and login:

    ssh jason@fedora
    export CR_PAT=<HIDDEN>
    echo $CR_PAT | docker login ghcr.io -u JasonGoemaat --password-stdin

Note: I get this message, may look into it:

    WARNING! Your password will be stored unencrypted in /home/jason/.docker/config.json.
    Configure a credential helper to remove this warning. See
    https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Create `docker-compose.yml` file on server:

```
services:
    sample:
        container_name: sample
        image: ghcr.io/jasongoemaat/docker-image-github:latest
        ports:
            - 3000:3000
```

Then start it:

    docker compose up -d

At this point it works, I can go to http://fedora:3000

Then I added this location to my `/etc/nginx/conf.d/goemaat.conf`:

    location /sample/ {
        proxy_pass "http://127.0.0.1:3000/";
    }

And I had to configure SELinux to allow nginx to connect externally or
I got a 502 bad gateway response:

    sudo setsebool -P httpd_can_network_connect 1

Then I can access the site with `https://goemaat.com/sample`

## Github Action

All files including `docker-compose.yml` and ones created here are stored on
fedora under `~/www/sample`.

Create ssh key - hitting ENTER for defaults (no passkey):

    ssh-keygen -t rsa -b 4096 -f ~/www/sample/id_rsa

Copy public hey to authorized_keys:

    cat ~/www/sample/id_rsa.pub >> ~/.ssh/authorized_keys

Copy private key (`id_rsa`) to clipboard:

    cat id_rsa

Go to github repository, then 'Settings' tab at top-right.  Then under
'Security', go to 'Secrets and variables', then 'Actions'.  Then click
'New repository secret'.   Use `SSH_PRIVATE_KEY` as the name and paste
the contents from `id_rsa` created above.   Then add another called
`SSH_USER` with the value `jason`.  Next is `SSH_HOST` for the address of
our server.

### Getting to run


### Update Fedora

I tried to upgrade fedora since my version was out of support...

    sudo dnf system-upgrade download --releasever=43 --allowerasing
    sudo dnf system-upgrade reboot

Somehow my btrfs partition became corrupt.   It would mount in readonly
mode and I tried for a few hours to fix it, to no avail.   I could boot and
access it in read-only mode, but the check command returned errors with inodes
and scrubbing/repairing did nothing.   I tried with a Fedora live cd so I could
mount it separately or unmount it and that didn't work either.   Finally
I copied the files I wanted to keep to a USB drive and just re-installed.  

This time I used a smaller partition, since shrink wouldn't work when there was
corruption.   That way I could create other partitions for other linux installs
if I wanted.  If this same problem happened again I could just install
Fedora into a new parition and copy the files at leasure since I could mount
it read-only with seemingly no problems other than btrfs commands not working.

I HATE THAT!   I've had a few drive failures in my 30+ years of using linux
(and Windows and MAC for that matter), but this is the first time I've had
some kind of file system corruption when the drive seems perfectly fine.  I'm
considering dropping btrfs for that reason.   It's supposed to be better than
other file systems in just this sort of scenario, but detecting errors doesn't
do my any good in my use case if there's no way to fix them.   I'd rather it
just let me delete or get rid of whatever was wrong.

    