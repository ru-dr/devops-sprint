## Docker

**Docker** is an open-source software platform which lets us package an application and anything its required to run, code, libraries, and settings into a single unit called **container**.

### Docker container

A **container** is an isolated process for each of the app's component.

Characteristics of container:
- Self-Contained - every container carries all the required things for it to function.
- Isolated - container run in isolation, so they have minimal influence on host and other container, for better security.
- Independent - each container is managed independently.
- Portable - container behaves the same in any machine no matter the specs.

### Docker Image

A **container Image** is a standard package which includes all the binaries, libraries, config and other dependencies to run a container. - *The Docker images are strictly **read-only** blocks.* 

Any changes we did in files stays in temporary, separate layer attached only to that specific container so if we change any file inside of image (the blueprint) it stays untouched.

### Multi-stage Dockerfile

A **Multi-stage build** uses multiple `FROM` statements. each indicates a separate stage.

Usually it follows similar structure.
- `stage 1` - Builder (*the part which install deps, run the build*)
- `stage 2` - Runner (*copy the build files - those whose only required and run it, keeping the runner lightweight*)

`COPY --from=builder /path /path` - copy the files from the one stage to another, here in this case from builder to runner.

### docker-compose

**docker-compose** use used to manage multiple containers from a one single file called - `.yml`. In that we define services (*each represent a different container*). Internally services talk with each other via service names using them as hostnames.

```yml
service:
    frontend:
        build:
            context: ./frontend
        ports:
            - "8000:8000"
    backend:
        build:
            context: ./backend
        ports:
            - "6000:6000"
    db:
        image: postgres:17
        environment:
            POSTGRES_USER: db_user
            POSTGRES_PASSWORD: db_pass
            POSTGRES_DB: main_db
```

run all with `docker compose up --build`

### Single-stage vs Multi-stage Comparison

**Single-stage build:**
- Everything in one `FROM` statement
- Simpler, but keeps all build tools in final image
- Build time: fast, Run time: slower (larger image to pull)

Example:
```dockerfile
FROM python:3.14
WORKDIR /app
COPY . .
RUN pip install fastapi uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
- Result: 1.66GB

**Multi-stage build:**
- Separate builder and runner stages
- Builder installs deps, runner copies only what's needed
- Strips out pip cache, dev libraries

Example:
```dockerfile
FROM python:3.14 AS builder
WORKDIR /app
COPY . .
RUN pip install --user fastapi uvicorn

FROM python:3.14-slim
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY --from=builder /app /app
ENV PATH=/root/.local/bin:$PATH
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
- Result: 199MB (8x smaller)

**When to use:**
- Single-stage: learning, small projects
- Multi-stage: production, size-sensitive deployments