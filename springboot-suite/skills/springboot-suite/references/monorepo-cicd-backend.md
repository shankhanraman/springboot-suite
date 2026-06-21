# Backend Reference Templates

Use these templates when generating backend files. Substitute all `{{VARIABLE}}` tokens
with the confirmed user inputs before writing.

Variables used in this file:
- `{{PROJECT_NAME}}` — e.g. `my-app`
- `{{BACKEND_PORT}}` — e.g. `8000`
- `{{AWS_REGION}}` — e.g. `us-east-1`
- `{{AWS_ACCOUNT_ID}}` — e.g. `123456789012`
- `{{DEPLOY_BRANCH}}` — e.g. `main`
- `{{LANGUAGE}}` — node / python / go
- `{{RUNTIME_VERSION}}` — e.g. `20` (node), `3.12` (python), `1.22` (go)
- `{{START_COMMAND}}` — e.g. `node dist/index.js`, `uvicorn main:app --host 0.0.0.0 --port 8000`
- `{{RUNTIME_ENVS}}` — list of env var names, e.g. `["DATABASE_URL", "JWT_SECRET"]`

---

## apps/backend/Dockerfile

### Node.js

```dockerfile
FROM node:{{RUNTIME_VERSION}}-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
RUN npm run build --if-present

FROM node:{{RUNTIME_VERSION}}-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
EXPOSE {{BACKEND_PORT}}
CMD ["{{START_COMMAND}}"]
```

### Python (FastAPI / Flask)

```dockerfile
FROM python:{{RUNTIME_VERSION}}-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE {{BACKEND_PORT}}
CMD ["{{START_COMMAND}}"]
```

### Go

```dockerfile
FROM golang:{{RUNTIME_VERSION}}-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

FROM alpine:latest AS runner
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE {{BACKEND_PORT}}
CMD ["./server"]
```

---

## apps/backend/.dockerignore

```
node_modules
.git
*.md
.env*
__pycache__
*.pyc
```

---

## infra/k8s/backend/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{PROJECT_NAME}}-backend
  namespace: {{PROJECT_NAME}}
  labels:
    app: {{PROJECT_NAME}}-backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: {{PROJECT_NAME}}-backend
  template:
    metadata:
      labels:
        app: {{PROJECT_NAME}}-backend
    spec:
      containers:
        - name: backend
          image: {{AWS_ACCOUNT_ID}}.dkr.ecr.{{AWS_REGION}}.amazonaws.com/{{PROJECT_NAME}}-backend:latest
          ports:
            - containerPort: {{BACKEND_PORT}}
          envFrom:
            - secretRef:
                name: {{PROJECT_NAME}}-backend-secrets
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: {{BACKEND_PORT}}
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: {{BACKEND_PORT}}
            initialDelaySeconds: 15
            periodSeconds: 10
```

Note: if `{{RUNTIME_ENVS}}` is empty/none, remove the `envFrom` block entirely.

Also generate `infra/k8s/backend/secret.yaml` if runtime envs were provided:

```yaml
# NOTE: Do not commit real values. Fill in values via kubectl or external-secrets.
apiVersion: v1
kind: Secret
metadata:
  name: {{PROJECT_NAME}}-backend-secrets
  namespace: {{PROJECT_NAME}}
type: Opaque
stringData:
  # Add each env var from {{RUNTIME_ENVS}} as a key here
  # Example:
  DATABASE_URL: "REPLACE_ME"
  JWT_SECRET: "REPLACE_ME"
```

---

## infra/k8s/backend/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{PROJECT_NAME}}-backend
  namespace: {{PROJECT_NAME}}
spec:
  selector:
    app: {{PROJECT_NAME}}-backend
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{BACKEND_PORT}}
  type: ClusterIP
```

Note: Backend uses `ClusterIP` (internal only). Only the frontend LoadBalancer is public-facing.

---

## .github/workflows/backend.yml

```yaml
name: Backend CI/CD

on:
  push:
    branches:
      - {{DEPLOY_BRANCH}}
    paths:
      - 'apps/backend/**'
      - '.github/workflows/backend.yml'

env:
  AWS_REGION: {{AWS_REGION}}
  ECR_REPOSITORY: {{PROJECT_NAME}}-backend
  IMAGE_TAG: ${{ github.sha }}

jobs:
  build-and-deploy:
    name: Build, push, and update manifest
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Run tests
        working-directory: apps/backend
        run: |
          # Node
          # npm ci && npm test
          # Python
          # pip install -r requirements.txt && pytest
          # Go
          # go test ./...
          echo "Add your test command here"

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        working-directory: apps/backend
        run: |
          IMAGE_URI=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
          docker build -t $IMAGE_URI .
          docker push $IMAGE_URI
          echo "IMAGE_URI=$IMAGE_URI" >> $GITHUB_ENV

      - name: Update image tag in K8s manifest
        run: |
          sed -i "s|image: .*{{PROJECT_NAME}}-backend:.*|image: ${{ env.IMAGE_URI }}|g" \
            infra/k8s/backend/deployment.yaml

      - name: Commit and push updated manifest
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add infra/k8s/backend/deployment.yaml
          git diff --staged --quiet || git commit -m "ci: update backend image to ${{ env.IMAGE_TAG }}"
          git push
```
