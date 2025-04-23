This repository contains information, scripts, and manifests for putting together a resource manager app, along with related information.

The resource manager has 4 parts, namely the Keycloak instance, the PostgreSQL database, the backend (housed in the `resource-manager-backend` repository), and the frontend (housed in the `resource-manager-frontend` repository). The Keycloak instance is to be run separately. The backend and frontend have their own separate repositories with instructions on how to run them locally as well as how to build a Docker image for them.

Docker
------

Once the Keycloak instance is up and configured (more info on how to configure it in the backend repository) and the backend and frontend images are built, the latter two along with the database can be started up with the `docker-compose.yml` file included in this repository, using the `docker compose up` command. For this, a `.env` file needs to be created containing necessary environment variables:
- `KEYCLOAK_URL`: The URL of the Keycloak instance, e.g. `http://192.168.1.134:8080/`
- `KEYCLOAK_ADMIN_USERNAME`: Username for admin of the master realm of the Keycloak instance, e.g. `admin`
- `KEYCLOAK_ADMIN_PASSWORD`: Password for admin of the master realm of the Keycloak instance, e.g. `admin`
- `KEYCLOAK_BACKEND_REALM`: Name of the realm to be used by the resource manager. Should be the same as for the frontend, however if it contains spaces this variable must have them replaced with `%20`, e.g. `Resource%20Manager`
- `KEYCLOAK_BACKEND_CLIENT`: Name of the client to be used by the backend, e.g. `resource-manager-backend`
- `KEYCLOAK_BACKEND_SECRET`: Secret string of the Keycloak realm, e.g. `f1jsLiEunf48R9a6yWnQu9dxbKwTKZ4j`
- `KEYCLOAK_FRONTEND_REALM`: Name of the realm to be used by the resource manager. Should be the same as for the backend, however if it contains spaces this variable must be surrounded in inverted commas, e.g. `"Resource Manager"`
- `KEYCLOAK_FRONTEND_CLIENT`: Name of the client to be used by the frontend, e.g. `resource-manager-frontend`
- `POSTGRES_USER`: Username of the PostgreSQL container, e.g. `admin`
- `POSTGRES_PASSWORD`: Password of the PostgreSQL container, e.g. `admin`
- `POSTGRES_URL`: Address of the PostgreSQL container including the port, e.g. `postgres:5432`
- `POSTGRES_DB`: Name of the PostgreSQL database, e.g. `resource-manager`
- `LOGOUT_URL`: Address to be used for generating the post-logout URL, e.g. `http://192.168.1.134:3000/`
- `PORT`: Port of the backend, to be used for starting the server and generating the post-logout URL, e.g. `3000`
- `LOGOUT_URL`: Complete path of the backend with trailing `/`, to be used by the frontend for API calls, e.g. `http://192.168.1.134:3000/`
- `FRONTEND_URL`: Complete path of the frontend with no trailing `/`, to be used by the backend for CORS as well as the frontend for generating the post-logout URL, e.g. `http://192.168.1.134`

A named volume, `postgres_data`, is used by the PostgreSQL container, mounted at `/var/lib/postgresql/data` for the purpose of keeping the database persistent through resets of the container.

To run in Swarm mode, a `docker-stack.yml` template has been provided. Since `docker stack` cannot automatically fill in environment variables as `docker compose` does, it is necessary to replace them before deploying the stack, either manually or with tools like `envsubst`. Once that is done, simply create and populate a swarm normally and deploy the stack with `docker stack deploy -c docker-stack.yml resource-manager --detach=false`.

For a small app as this one, simply running it with Docker Compose on one machine might be enough. If scaling and high availability is necessary, Docker Swarm would be the simpler solution, with the caveat that it requires using an external solution for sharing the database across nodes. Kubernetes has better support for this issue, though it adds significant complexity for the deployment and would be recommended only if more of its functions, such as advanced networking and security, are necessary.

Kubernetes
----------

To set up the resource manager as part of a Kubernetes cluster, follow the instructions below.

Basic setup:
- Set up k3s with Traefik and Cert-Manager in their own dedicated `traefik` and `cert-manager` namespaces.
- Create a secret containing the `api-token` key with the certificate issuer API token as its value.
- Apply the given `kubernetes/cluster-issuer.yaml` template, replacing the name `CLUSTER_ISSUER` as well as the credentials `CERT_EMAIL`, `CERT_SERVER` and `CERT_TOKEN` with environment variables, envsubst, manually, or by implementing a secret (**the same will apply for any later substitutions in the templates**).

Database setup:
- Create a `postgresql` namespace with a `postgresql-secret` containing the `postgres-password` key with the chosen PostgreSQL superuser password as value
- From the `kubernetes/postgresql` directory, apply the `postgresql-pv.yaml` and `postgresql-pvc.yaml` manifests to create the persistent volume and claim for the database, then use the `postgresql-values.yaml` manifest to deploy the database with the Postgres Bitnami Helm chart, all in the dedicated namespace.
- Enter the PostgreSQL pod as superuser and create separate users for the Keycloak and resource manager backend with the following commands as a model, replacing the pod name, credentials, and database names as needed:
```
kubectl exec -it postgresql-0 -n postgresql -- psql -U postgres
CREATE USER kc WITH LOGIN PASSWORD 'kc';
CREATE USER rm WITH LOGIN PASSWORD 'rm';
CREATE DATABASE kcdb OWNER kc;
CREATE DATABASE rmdb OWNER rm;
```

Keycloak setup:
- Create a `keycloak` namespace and apply the `kubernetes/keycloak/keycloak-ingress.yaml` manifest in it, replacing `CLUSTER_ISSUER` with the name of the one created earlier and `KEYCLOAK_HOST` with the domain that Keycloak will be hosted on.
- Deploy Keycloak in the namespace with the Bitnami Helm chart using the `kubernetes/keycloak/keycloak-values.yaml` manifest, replacing `KEYCLOAK_ADMIN_USER` and `KEYCLOAK_ADMIN_PASSWORD` with the initial admin credentials, and `KEYCLOAK_POSTGRES_USER`, `KEYCLOAK_POSTGRES_PASSWORD`, and `KEYCLOAK_POSTGRES_DATABASE` with the username, password, and database name for Keycloak created in the pod.
- More info on how to configure Keycloak in the web console to work with the backend and frontend can be found in their respective repositories.

App setup:
- Create a `resource-manager` namespace containing the secrets `backend-env` and `frontend-env`. Templates are provided in `kubernetes/backend/backend-env.yaml` and `kubernetes/frontend/frontend-env.yaml` respectively, and further information on the variables to fill in can be found in the repositories of the respective components or the Docker section above.
- The Ingress can be set up by applying `backend-ingress.yaml` and `frontend-ingress.yaml` after filling in `CLUSTER_ISSUER` as with the Keycloak Ingress, as well as `BACKEND_HOST` and `FRONTEND_HOST` (for the domains on which to deploy the respective components),
- Finally, the app itself can be deployed by applying `backend-deploy.yaml` and `frontend-deploy.yaml` after filling in `BACKEND_IMAGE_URL` and `FRONTEND_IMAGE_URL` (for the URLs from which to pull the respective images).

CI/CD pipeline setup:
- The backend and frontend repositories have manifests for optionally creating an automated GitLab CI/CD pipeline. For this, they need to be cloned and uploaded on a GitLab account which has at least one usable runner set up as well as the variables `CI_RUNNER_TAG` (for the tag of the runner(s) to be used), `CI_REGISTRY_PATH`, `CI_REGISTRY_PROJECT`, `CI_REGISTRY_USER` and `CI_REGISTRY_PASSWORD` (for the base path of the image registry, the project name on the registry, and the credentials of the registry user or robot to push and pull with), `CI_DOMAIN` (for the domain to be deployed on), and `KUBECONFIG_dATA` (for the configuration data of the cluster obtained with `cat ~/.kube/config` that **must be encoded with base64**). These variables will be used to fill in the ingress and deploy manifests in the repos. The secrets that provide the rest of the variables that determine the connections of the backend and frontend to the database, keycloak, and each other have to already exist in the cluster.
- The pipeline is generalised for multiple branches/environments. The `main` branches will simply use `backend` and `frontend` for the names of the images and `backend-env` and `frontend-env` for the names of the secrets respectively, and will be deployed automatically. For other branches, the pipeline needs to be triggered automatically, and will add the name of the branch as a suffix to name of the image and domain, e.g. `backend-dev:latest` and `backend-dev.yourdomain.com` as well as expect a secret with that suffix to already exist in the cluster, e.g. `backend-env-dev`.

MinIO (backup and restore) setup:
- An optional backup and restore mechanism for the database vis MinIO has been provided. For this, a `minio` namespace needs to be created with a `minio-secret` which contains the keys `root-user` and `root-password` for the root user credentials (**warning, the password must be at least 8 characters long**).
- A persistent volume and claim must be created by applying the `minio.pv.yaml` and `minio-pvc.yaml` manifests from `kubernetes/minio` (by default, 8Gi is allocated, but it can be changed by editing the manifests).
- The Ingress can be created with the `minio-ingress.yaml` manifest from the same directory after filling in the `CLUSTER_ISSUER` and `MINIO_HOST` as in the previous ingresses, and the MinIO server can be deployed using the Bitnami Helm chart with the `minio-deploy.yaml` manifest also from the same directory.
- To implement automated backup, apply the `kubernetes/backup.yaml` manifest after filling in the old variables `KEYCLOAK_POSTGRES_USER`, `KEYCLOAK_POSTGRES_PASSWORD`, `KEYCLOAK_POSTGRES_DATABASE` (to access the Keycloak database), `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` (to access the resource manager database) if they aren't already set, as well as the new variables `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD` (for the credentials of the MinIO root user set earlier). This will create a Cron Job that activates at 02:00 UTC (by default, can be changed in the `schedule` key of the manifest), creates an ephemeral volume, creates a PostgreSQL container to dump and timestamp the databases, creates a MinIO client container to create buckets if they don't already exist then upload the files to the MinIO server, and finally deletes the containers and volume.
- To manually execute the backup immediately, simply create a normal Job from the Cron Job with e.g. `kubectl create job backup-manual --from=cronjob/backup`.
- To restore the database from the latest backup, simply apply the `kubernetes/restore.yaml` manifest after filling in the same variables as in the backup manifest. This essentially runs the backup process in reverse by creating a MinIO client container fetching a list of the buckets for each backup, sorting them by date modified to find the latest ones then downloading them to an ephemeral volume, after which a PostgreSQL container is created to drop the existing public schemas and push the downloaded backups as replacement.
