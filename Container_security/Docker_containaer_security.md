# 🔐 DevSecOps Container Security – Best Practices

This repository demonstrates **container security best practices for Docker** as part of a DevSecOps workflow.

The examples use a simple **NodeJS application** and show how to improve container security step-by-step.

Topics covered:

1. Running containers as **non-root user**
2. **Multi-stage Docker builds**
3. **Distroless container images**
4. Using **.dockerignore**
5. **Runtime container hardening**

---

## Sample Application

The demo application is a simple **NodeJS Hello World app**.

It returns:

- Hello from container
- Container hostname
- UID of the running user

The application runs on **port 3000**.

### Run Application Locally

```bash
npm install
npm start
```


#### 1️⃣ Insecure Dockerfile (Running as Root)

A common mistake by beginner DevOps engineers is creating a Dockerfile that runs containers as root user.

Dockerfile Example
```bash
FROM node:25

WORKDIR /app

COPY app.js package.json ./

RUN npm install

EXPOSE 3000

CMD ["npm","start"]
```
Build Image
```bash
docker build -t insecure-image .
```

Run Container
```bash
docker run -p 3000:3000 insecure-image
```

Verify
curl localhost:3000

You will notice: UID = 0
UID 0 means the container is running as root user, which is a security risk.

## Security Risks of Root Containers

Running containers as root can lead to:

- • Privilege escalation
- • Host system compromise
- • Resource abuse (DOS attacks)
- • Unauthorized package installation
- • Access to Docker daemon

If an attacker gains container access, they may also gain access to the host machine.

#### 2️⃣ Running Container as Non-Root User

To improve security, create a dedicated user inside the container.

Secure Dockerfile
```bash
FROM node:25

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY app.js package.json ./

RUN npm install

USER appuser

EXPOSE 3000

CMD ["npm","start"]
```

Follow the above similar steps to build and verify 
Verify User
id
Output: uid=999(appuser)

Now the container runs as non-root user.

### 3️⃣ Multi-Stage Docker Builds

Even after switching to non-root users, the image size may still be large.

Example image size: 401 MB

Large images contain:

• Unnecessary dependencies
• Extra build tools
• Vulnerable packages

Multi-stage builds solve this.

Multi-Stage Dockerfile
```bash
FROM node:25 AS builder

WORKDIR /build

COPY package.json ./

RUN npm install

COPY app.js .

FROM node:25-slim

WORKDIR /app

COPY --from=builder /build .

EXPOSE 3000

CMD ["node","app.js"]
```
Build Image
```bash
docker build -t multi-stage-image .
```

Image Size  ~80 MB
This removes unnecessary build dependencies from the runtime image.

Benefits of Multi-Stage Builds

• Smaller image size
• Reduced attack surface
• Faster deployments
• Fewer vulnerable packages

### 4️⃣ Distroless Images

Even slim images contain unnecessary system utilities such as:
- apt
- shell tools
- system binaries

These increase attack surface. Distroless images remove unnecessary OS components.

Distroless Dockerfile
```bash
FROM node:25 AS builder

WORKDIR /build

COPY package.json ./

RUN npm install

COPY app.js .

FROM gcr.io/distroless/nodejs20-debian12

WORKDIR /app

COPY --from=builder /build .

EXPOSE 3000

CMD ["app.js"]
```
Image Size ~52 MB

Benefits of Distroless Images

• Minimal system binaries
• Reduced attack surface
• Smaller image size
• Better container security

5️⃣ .dockerignore

Just like .gitignore, Docker supports .dockerignore.

Without it, Docker may copy unnecessary files like:
- .git
- node_modules
- .env
- build artifacts

Example .dockerignore
.git
node_modules
Dockerfile
README.md
.env

This prevents unnecessary files from being included in the container image.

6️⃣ Container Runtime Hardening

Security is not only about Dockerfiles.

Runtime configuration also matters.

Secure Docker Run Example
```bash
docker run \
--read-only \
--tmpfs /tmp \
--cap-drop ALL \
--security-opt no-new-privileges \
--pids-limit 100 \
--memory 512m \
--cpus 1 \
-p 3000:3000 \
distroless-image
```

Runtime Security Parameters
Read Only Filesystem

--read-only

Temporary Filesystem
--tmpfs /tmp

Allows temporary file storage.

Drop Linux Capabilities
--cap-drop ALL

Removes unnecessary privileges.

Disable Privilege Escalation
--security-opt no-new-privileges

Prevents processes from gaining elevated permissions.

Limit Processes
--pids-limit 100

Prevents DOS attacks from process spawning.

Limit Resources
--memory
--cpus

Prevents containers from exhausting host resources.

### Final DevSecOps Container Security Checklist
| Security Control          | Purpose                          |
| ------------------------- | -------------------------------- |
| Run container as non-root | Prevent privilege escalation     |
| Multi-stage builds        | Reduce attack surface            |
| Distroless images         | Remove unnecessary binaries      |
| .dockerignore             | Prevent sensitive file inclusion |
| Runtime hardening         | Limit container capabilities     |
