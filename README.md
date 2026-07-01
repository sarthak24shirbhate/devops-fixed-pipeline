# DevOps Troubleshooting Assignment

## What this was about

This assignment came with a handful of bugs baked into the project on purpose — spread across Docker, Docker Compose, Kubernetes, Nginx, and the app's own health check. The point wasn't just to make things "work" by trial and error, but to actually track down why each piece was broken and fix the real cause. Below is a walkthrough of what I found and how I fixed it, roughly in the order I worked through them.

---

## 1. The Docker image wouldn't build

First thing I tried was `docker build`, and it failed almost immediately. Looking at the Dockerfile, it was trying to do this:

```dockerfile
COPY src .
```

Except there's no `src` folder anywhere in the repo — `server.js` and `package.json` just sit directly inside `app/`. So the build was choking on a path that didn't exist. Fixed it by pointing the COPY at the current build context instead:

```dockerfile
COPY . .
```

Once that changed, the image built without complaint.

## 2. `/health` was lying about the app's health

This one was almost funny once I saw it:

```javascript
app.get('/health', (req, res) => res.status(500).send("NOT OK"));
```

The endpoint was hardcoded to return a 500 no matter what. That's a problem because Kubernetes, load balancers, and pretty much any monitoring setup rely on this endpoint to decide whether to send traffic to the pod at all. With this in place, the app would get treated as unhealthy forever, even when it was running just fine. Swapped it for an actual healthy response:

```javascript
app.get('/health', (req, res) => res.status(200).send("OK"));
```

## 3. Compose couldn't find anything to build

`docker-compose up` failed at the build step because the compose file was looking in the wrong place:

```yaml
build: ./backend
```

There's no `backend` directory in this repo — the app lives in `app/`. Changed it to:

```yaml
build: ./app
```

and Compose was able to find the Dockerfile without issue.

## 4. The app couldn't talk to Postgres

This one had two separate problems stacked on top of each other, which made it a bit more annoying to isolate.

First, the hostname was wrong. Compose gives you internal DNS based on service names, and the Postgres service in this project is just called `db`. But the app was configured to reach out to `dbserver`, which doesn't exist anywhere in the compose file:

```yaml
DATABASE_HOST: dbserver
```

Second, the port was off too. Postgres defaults to `5432`, but the app was set to:

```yaml
DATABASE_PORT: 5433
```

Both got corrected:

```yaml
DATABASE_HOST: db
DATABASE_PORT: 5432
```

After that, the connection came up clean.

## 5. Nginx was throwing 502s

Hitting the app through Nginx gave a 502 Bad Gateway every time. The proxy config explained why:

```nginx
proxy_pass http://application:3001;
```

Two things wrong here — there's no service in the stack named `application` (it's just `app`), and the Node app listens on port `3000`, not `3001`. Updated the proxy target:

```nginx
proxy_pass http://app:3000;
```

Requests started flowing through to the backend properly after that.

## 6. Kubernetes pod stuck in CrashLoopBackOff

Last one, and it also turned out to be two bugs in one manifest.

The deployment was trying to start the container with:

```yaml
command:
- node
- index.js
```

but there's no `index.js` in this project — the entry point is `server.js`. On top of that, the liveness probe was checking a path that doesn't exist on the app at all:

```yaml
path: /live
```

The actual health route (see issue #2 above) is `/health`. Fixed both:

```yaml
command:
- node
- server.js

livenessProbe:
  httpGet:
    path: /health
```

With the right entry point and the right probe path, the pod started up and stayed running.

---

## Assumptions I made along the way

- The `app-secret` Kubernetes Secret already exists in the cluster — I didn't create it myself since it wasn't part of the broken pieces.
- Docker Desktop / Docker Engine is up and running before anyone runs Compose.
- Postgres is left on its default port (5432), nothing custom.

---

## A few things I'd improve if this were a real production setup

None of these were required to fix the assignment, but they stood out while I was in there:

- Add a `.dockerignore` — the image is pulling in more than it needs to right now.
- Swap `npm install` for `npm ci` so builds stay reproducible.
- Add a `readinessProbe` alongside the existing `livenessProbe` — right now Kubernetes only knows if the app is alive, not if it's actually ready to serve traffic.
- Set CPU/memory requests and limits on the container instead of leaving them unbounded.
- Stop using the `latest` tag for anything that goes to production — makes rollbacks and debugging harder than they need to be.
- Move secrets into Kubernetes Secrets or Docker Secrets properly rather than any hardcoded values.
- A basic GitHub Actions workflow for build validation would catch a lot of this automatically next time.

---

## Files I touched

```
app/Dockerfile
app/server.js
docker-compose.yml
kubernetes/deployment.yaml
nginx/nginx.conf
README.md
```

---

## One thing I couldn't do

The assignment brief mentions a failing GitHub Actions pipeline, but there's no `.github/workflows` directory anywhere in the repo I was given. So I wasn't able to look into or fix anything CI/CD related — there just wasn't a workflow to debug.

---

## Wrap-up

Went through each reported issue one at a time, figured out what was actually causing it rather than guessing, and made the minimal change needed to fix it. Everything here touches Docker's build setup, Compose's service config, the Postgres connection, the Nginx reverse proxy, the Kubernetes deployment, and the app's health check — nothing outside of that was touched.

---

# Local Validation

After applying all the fixes, I wanted to make sure the project was actually working instead of assuming the changes were correct. I rebuilt the project from scratch and verified each component locally using Docker Desktop.

## Build Verification

I rebuilt the application using Docker Compose.

```bash
docker compose build
```

The build completed successfully without any Docker build errors, confirming that the Dockerfile and build context were correctly configured.

### Docker Build Result

![Docker Build Successful](docker%20build%20image%20.jpg)

---

## Running the Application

After the build completed successfully, I started the complete application stack.

```bash
docker compose up
```

Docker Compose successfully created and started the required containers:

- Node.js Application
- PostgreSQL Database
- Nginx Reverse Proxy

The application was then accessed through the Nginx reverse proxy using:

```
http://localhost
```

The expected response was returned successfully.

### Running Application

![Application Running Successfully](pipeline%20working%20successfully%20.jpg)

---

## Verification Summary

During testing, I verified the following:

- Docker image builds successfully.
- Docker Compose starts all required services.
- PostgreSQL initializes successfully.
- Nginx proxies requests correctly.
- The Node.js application is reachable through Nginx.
- Accessing `http://localhost` returns the expected **"Hello"** response.

This confirms that the issues provided in the assignment were successfully identified, resolved, and validated in a local environment.

---

## Notes

During the first application startup, I observed a temporary database connection error (`ECONNREFUSED`) because the application attempted to connect before PostgreSQL had fully completed its initialization. Once PostgreSQL finished starting, all services became available and the application worked as expected.

For a production deployment, I would further improve this by adding:

- PostgreSQL health checks
- `depends_on` with `service_healthy`
- Database connection retry logic in the application

These improvements would make the application more resilient during container startup.



