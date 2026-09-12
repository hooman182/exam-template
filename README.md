# Scenario 1

## Problem 1: Python Virtual Environment

**What was wrong:**

Creating the Python virtual environment initially failed because the `python3-venv` package was not installed.

**How I fixed it:**

I installed the required package and created the virtual environment again.

**Config I changed (only the changed part):**

No configuration file was changed.

**Commands I used:**

```bash
python3 -m venv .venv

apt update
apt install -y python3-venv

python3 -m venv .venv
source .venv/bin/activate
```

---

## Problem 2: DNS Resolution

**What was wrong:**

The server initially could not resolve external domains. Because of this, `apt` and other network commands could not connect to external servers.

**How I fixed it:**

I started and enabled `systemd-resolved` and corrected the DNS resolver configuration.

**Config I changed (only the changed part):**

```text
/etc/resolv.conf
```

**Commands I used:**

```bash
systemctl status systemd-resolved
systemctl start systemd-resolved
systemctl enable systemd-resolved

cat /etc/resolv.conf

ping google.com
getent hosts pypi.org
apt update
```

---

## Problem 3: PyPI Connection Timeout

**What was wrong:**

After fixing DNS, `pip install -r requirements.txt` still failed because the connection to `pypi.org` timed out during the TLS handshake.

I tested other Python infrastructure and found that `files.pythonhosted.org` was reachable. This indicated a connectivity problem specifically with the default PyPI server.

**How I fixed it:**

I used an accessible PyPI mirror instead of the default PyPI repository.

**Config I changed (only the changed part):**

No project configuration was changed.

**Commands I used:**

```bash
pip install -r requirements.txt
```

After the timeout:

```bash
pip install -r requirements.txt -i https://mirror-pypi.runflare.com/simple
```

---

## Problem 4: Backend Could Not Resolve PostgreSQL

**What was wrong:**

After starting the Docker containers, the backend could not connect to PostgreSQL.

The backend logs showed:

```text
psycopg2.OperationalError:
could not translate host name "db" to address:
Name or service not known
```

I first suspected that PostgreSQL was not ready yet. After checking the Docker networks, I found that the `backend` and `db` containers were on different Docker networks.

**How I fixed it:**

I connected the backend to both the Nginx-backend network and the backend-database network.

**Config I changed (only the changed part):**

```yaml
backend:
  ...
  networks:
    - nginx-backend-net
    - backend-db-net
```

**Commands I used:**

```bash
docker compose ps

docker compose logs backend

docker inspect service-catalog-backend-1
docker inspect service-catalog-db-1

docker network inspect backend-db-net

docker compose exec backend getent hosts db
```

After changing the network configuration:

```bash
docker compose up -d
```

Then I verified DNS resolution:

```bash
docker compose exec backend getent hosts db
```

Result:

```text
172.18.0.2 db
```

I also tested the actual database connection:

```bash
docker compose exec backend python3 -c "import psycopg2; psycopg2.connect('postgresql://catalog:catalog@db:5432/catalog'); print('DATABASE CONNECTION OK')"
```

Result:

```text
DATABASE CONNECTION OK
```

---

## Problem 5: Incorrect Nginx Backend Address

**What was wrong:**

Nginx was configured to forward requests to:

```text
backend-api:8080
```

But the actual Docker Compose service was named `backend`, and Gunicorn was listening on port `5000`.

**How I fixed it:**

I changed the Nginx upstream to use the correct Docker service name and port.

**Config I changed (only the changed part):**

```nginx
set $backend_upstream http://backend:5000;
```

**Commands I used:**

```bash
cat nginx/nginx.conf

docker compose config

docker compose exec nginx getent hosts backend

docker compose logs nginx

docker compose restart nginx
```

I verified that Nginx could resolve the backend:

```bash
docker compose exec nginx getent hosts backend
```

Result:

```text
172.19.0.2 backend backend
```

I then tested the application through Nginx:

```bash
curl http://127.0.0.1/graph
```

The endpoint returned the expected graph JSON, confirming that Nginx could reach the Flask backend.

---

# Extra problems

* **Missing `python3-venv` package ** — The Python virtual environment could not be created because the required package was not installed.

* **DNS configuration ** — External domains could not initially be resolved, so `apt` and other network commands failed.

* **PyPI connectivity ** — The default PyPI server timed out, so I tested connectivity and used an alternative mirror.

* **Docker Compose command/package ** — The Docker Compose command was initially unavailable, so the required Compose package was installed.

* **Initial PostgreSQL diagnosis ** — I first suspected PostgreSQL startup timing, but inspection showed that the real problem was the Docker network configuration.

* **Testing the wrong `/` endpoint ** — I initially tested `/`, which returned `404`. I then checked the application routes and found that `/graph` was the intended endpoint.
