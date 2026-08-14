Tictactoe (mircroserviced architechture)


## Assignment checklist

Mircroservice Architechture is applied.
Both at Frondend and Backend.
The user profile picture is stored in Azure Blob Storage.(avatars uploaded to the container in blob storage successfully).
Rest API is USED to communicate between frontend and backend.
Docker is used to create containers.
Minikube is used to orchestrate Containers
the whole website is hosted in azure container apps. (There were some resources not available so the the docker images are uploaded to cloud and then with the help of these images the website is now live.) No service is serverless because of limited azure cloud resources and region unavailability problem.


- **independently deployable backend services** 
   each service in the backend has its own .json file instead of database.
- **REST only.** The frontend only ever talks to `api-gateway`; the gateway
  proxies each request to the right service; `game-service` calls
  `leaderboard-service` over REST when a match finishes. 
  No shared code, no shared database, no direct service-to-service imports.
- **Cloud storage.** Avatar images uploaded from the Profile page are sent
  through `auth-service` to an **Azure Blob Storage** container
  named as "avatars".
  If no Azure connection string is configured, it falls back to local disk so you can
  develop without a cloud account.


## Running it locally (no Docker, no Kubernetes)

The simplest way to see the project working.

Each service needs `npm install` once, then a `.env` file copied from its
`.env.example`:

```bash
# from the repo root, for each service:
cd services/auth-service        && cp .env.example .env && npm install
cd ../game-service               && cp .env.example .env && npm install
cd ../leaderboard-service        && cp .env.example .env && npm install
cd ../api-gateway                && cp .env.example .env && npm install
cd ../../frontend                && cp .env.example .env && npm install
```

Then start all five apps, each in a different terminal.
This makes the total number of terminals 5 running to make the game playable.

```bash
cd services/auth-service        && npm start   # :4001
cd services/game-service        && npm start   # :4002
cd services/leaderboard-service && npm start   # :4003
cd services/api-gateway         && npm start   # :4000
cd frontend                     && npm run dev # :5173
```

Open http://localhost:5173.


## Running it with Docker Compose

The same five apps, but each one in its own container, wired together
automatically.

```bash
# (do this only if you didnot try the first method).
# copy every .env.example to .env first (same as above line37 to 41), then:
docker compose up --build
```

This builds and starts all five containers (see `docker-compose.yml`).
Open http://localhost:5173 once it finishes starting. or form the docker app click on the link infornt of the frontend container.

## Running it with Kubernetes (minikube)

This is the container-orchestration requirement: instead of Docker Compose
starting five containers on one machine, Kubernetes runs five **Deployments**
(self-healing groups of containers) and wires them together with
**Services** (stable network names), all inside a small local cluster called
minikube. Everything Kubernetes needs is in the `k8s/` folder, one file per
piece so it's easy to read.

# if you have these then you dont need to install. (probably you will have them)
**One-time setup** — install [Docker](https://docs.docker.com/get-docker/),
[minikube](https://minikube.sigs.k8s.io/docs/start/) and
[kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) 

**1. Start the cluster**

```bash
minikube start
```

**2. Build the five container images *inside* minikube**

Minikube runs its own little Docker, separate from the normal `docker`
command, so images need to be built specifically into it — otherwise
Kubernetes won't be able to find them.

```bash
minikube image build -t tictactoe-hub/auth-service:latest        ./services/auth-service
minikube image build -t tictactoe-hub/game-service:latest        ./services/game-service
minikube image build -t tictactoe-hub/leaderboard-service:latest ./services/leaderboard-service
minikube image build -t tictactoe-hub/api-gateway:latest         ./services/api-gateway
minikube image build -t tictactoe-hub/frontend:latest             ./frontend
```

**3. Create your secrets file**

`k8s/02-secret.example.yaml` is a template (safe to commit — it has no real
secrets in it). Copy it and fill in real values:

```bash
cp k8s/02-secret.example.yaml k8s/02-secret.yaml
```

Then edit `k8s/02-secret.yaml`: replace `JWT_SECRET` with a base64-encoded
random string (`echo -n "some-long-random-string" | base64`), and, if you
have Azure Blob Storage set up, base64-encode your connection string into
`AZURE_STORAGE_CONNECTION_STRING` the same way. Leaving it blank makes
`auth-service` fall back to local disk, same as running it without Docker.
 do the following commands and paste accordingly as mentioned above:
## echo -n "some-long-random-string" | base64
## echo -n "azure connection string shoould be pasted here" | base64


**4. Apply everything**

```bash
kubectl apply -f k8s/
```
you are done, just check every thing and you can access my website.

**5. Check that everything is running**

```bash
kubectl get pods -n tictactoe-hub
```

Wait until every pod shows `STATUS: Running` and `READY: 1/1` (this can take
a minute the first time). If a pod is stuck, 
`kubectl logs -n tictactoe-hub <pod-name>` shows why.

**6. Open the app**

```bash
minikube service frontend -n tictactoe-hub --url
minikube service api-gateway -n tictactoe-hub --url
```

## this is very important as the url to api is the main gateway.
The first command gives you the URL to open in your browser. The second
gives you the gateway's URL — if it isn't `http://localhost:4000`, update
`VITE_API_BASE_URL` in `k8s/01-configmap.yaml` to match, then re-run
`kubectl apply -f k8s/01-configmap.yaml` and restart the frontend pod
(`kubectl rollout restart deployment/frontend -n tictactoe-hub`).


**Cleaning up**

```bash
kubectl delete -f k8s/
minikube stop
```





till here, the above commands are only to run it locally.
## Below you can see some thing you need to do while hosting it in a cloud.







## Setting up Azure Blob Storage (cloud requirement)

1. In the Azure Portal, create a **Storage Account** (Standard, LRS is fine
   for a class project).
2. Inside it, you don't need to pre-create a container — the service calls
   `createIfNotExists` on first upload — but you can create one named
   `avatars` if you'd rather do it by hand.
3. Go to **Storage Account → Security + networking → Access keys**, copy a
   **connection string**.
4. Paste it into `services/auth-service/.env` as
   `AZURE_STORAGE_CONNECTION_STRING=...` (or, for Kubernetes, base64-encode
   it into `k8s/02-secret.yaml` — see step 3 above).
5. Restart `auth-service`. Uploading an avatar from the Profile page will now
   write to Blob Storage and store the public blob URL on the user record.

**One-time setup**

1. Install the [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli).
2. Log in: `az login` (opens a browser to sign in).
3.  Docker should be installed and running locally —
   As images are built on your machine and pushed to Azure, since 
   my free student subscriptions don't allow ne to build them in the cloud.


## Deploying to Azure

this command does everything. as we are using the docker images and pushing them to cloud so the command 
below runs the ./azure/deploy.sh and does all the work:

AZURE_STORAGE_CONNECTION_STRING="`My azure connection string`" LOCATION="polandcentral" ./azure/deploy.sh


or this command works as well if the connection string is already given.

```bash
cd tictactoe-hub
./azure/deploy.sh
```

The script prints progress as it goes and, at the end, prints the public
URL to open. In order, it:


here is the public url:

https://frontend.greensky-d59f8e5e.polandcentral.azurecontainerapps.io


It is abit slow but it works fine....



Thank you for reading this long.
