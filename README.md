# bombur-deploy

Deploy tooling for apps on Bombur (`bombur.remesh.cz`): GitHub Actions builds an image to
GHCR, then asks Bombur to run a given SHA. CI can do nothing else on the server.

```
GitHub repo ──(own key, environment secret)──► ssh deploy@bombur:10022 "deploy <sha>"
                                                  │ authorized_keys forced command
                                                  ▼
                                  /usr/local/bin/bombur-deploy <app>  (root-owned)
                                    validates the SHA, sets IMAGE_TAG in ~deploy/DOCKER/<app>/.env,
                                    docker compose pull && up -d
```

The server owns `compose.yaml`, `.env` and the image name; CI only picks the tag.

| File | Runs on | Purpose |
|---|---|---|
| `bin/bombur-deploy` | Bombur, as `deploy` (forced command) | The only thing a CI key can run |
| `sbin/bombur-add-app` | Bombur, as root via sudo | Creates the app dir/`.env`, installs the deploy key, adds the Caddy site |
| `bin/bombur-onboard` | your machine, in the app repo | Generates a key, calls `bombur-add-app`, sets the GitHub environment secret |
| `.github/workflows/deploy.yml` | GitHub Actions | Reusable deploy job (`uses: ciakval/bombur-deploy/.github/workflows/deploy.yml@v1`) |

## New app

1. In the app repo: Dockerfile, `compose.yaml` (container `<app>-web`, on the `proxy`
   network, image `ghcr.io/${GITHUB_REPOSITORY}:${IMAGE_TAG}`), `.env.example` (use
   `CHANGE_ME` for secrets to be generated), and a workflow that builds the image and calls
   the reusable deploy (example in `deploy.yml`'s header).
2. `bombur-onboard <app> <port>` from the repo root. DNS is covered by `*.remesh.cz`.
3. Fill real secrets: `ssh -p 10022 ciakval@bombur.remesh.cz sudo -u deploy nano /home/deploy/DOCKER/<app>/.env`
4. Protect `main` (at least no force-push) so the `production` environment accepts it, then push.

Re-running `bombur-onboard` refreshes `compose.yaml` on the server and rotates the key;
`BOMBUR_KEY_ONLY=1` rotates the key only. Pass a third argument to use another GitHub
environment (e.g. `bombur-onboard myapp-test 8000 test`).

## Operations

```sh
ssh -p 10022 ciakval@bombur.remesh.cz
cd /home/deploy/DOCKER/<app>
sudo -u deploy docker compose ps / logs -f
journalctl -t bombur-deploy            # deploy history
# Rollback: re-run an older successful workflow run in GitHub, or
sudo -u deploy sed -i 's/^IMAGE_TAG=.*/IMAGE_TAG=<old-sha>/' .env && sudo -u deploy docker compose up -d
```

Revoke an app's CI access: delete its `gh-<app>` line from `/home/deploy/.ssh/authorized_keys`.

## Installing on Bombur

```sh
sudo install -o root -g root -m 755 bin/bombur-deploy /usr/local/bin/
sudo install -o root -g root -m 755 sbin/bombur-add-app /usr/local/sbin/
echo 'ciakval ALL=(root) NOPASSWD: /usr/local/sbin/bombur-add-app' | sudo tee /etc/sudoers.d/bombur-add-app
```

`bin/bombur-onboard` goes on your `PATH` locally.
