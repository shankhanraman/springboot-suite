# Frontend Reference Templates

Use these templates when generating frontend files. Substitute all `{{VARIABLE}}` tokens
with the confirmed user inputs before writing.

Variables used in this file:
- `{{PROJECT_NAME}}` — e.g. `my-app`
- `{{NODE_VERSION}}` — e.g. `20`
- `{{BUILD_COMMAND}}` — e.g. `npm run build`
- `{{FRONTEND_PORT}}` — e.g. `3000`
- `{{AWS_REGION}}` — e.g. `us-east-1`
- `{{AWS_ACCOUNT_ID}}` — e.g. `123456789012`
- `{{DEPLOY_BRANCH}}` — e.g. `main`
- `{{GITHUB_ORG}}` — e.g. `acme-corp`
- `{{API_URL_ENV}}` — e.g. `NEXT_PUBLIC_API_URL` (optional — omit ARG block if not provided)
- `{{FRAMEWORK}}` — Next.js / React / Vue / plain HTML

---

## apps/frontend/Dockerfile

### Next.js (multi-stage, production optimised)

```dockerfile
FROM node:{{NODE_VERSION}}-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci --only=production

FROM node:{{NODE_VERSION}}-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
{{#if API_URL_ENV}}
ARG {{API_URL_ENV}}
ENV {{API_URL_ENV}}=${{API_URL_ENV}}
{{/if}}
RUN {{BUILD_COMMAND}}

FROM node:{{NODE_VERSION}}-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./package.json
EXPOSE {{FRONTEND_PORT}}
CMD ["node_modules/.bin/next", "start", "-p", "{{FRONTEND_PORT}}"]
```

### React / Vue / plain HTML (nginx static serve)

```dockerfile
FROM node:{{NODE_VERSION}}-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
{{#if API_URL_ENV}}
ARG {{API_URL_ENV}}
ENV {{API_URL_ENV}}=${{API_URL_ENV}}
{{/if}}
RUN {{BUILD_COMMAND}}

FROM nginx:alpine AS runner
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE {{FRONTEND_PORT}}
CMD ["nginx", "-g", "daemon off;"]
```

Include `nginx.conf` for React/Vue:
```nginx
server {
    listen {{FRONTEND_PORT}};
    root /usr/share/nginx/html;
    index index.html;
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## apps/frontend/.dockerignore

```
node_modules
.next
.git
*.md
.env*
```

---

## infra/k8s/frontend/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{PROJECT_NAME}}-frontend
  namespace: {{PROJECT_NAME}}
  labels:
    app: {{PROJECT_NAME}}-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: {{PROJECT_NAME}}-frontend
  template:
    metadata:
      labels:
        app: {{PROJECT_NAME}}-frontend
    spec:
      containers:
        - name: frontend
          image: {{AWS_ACCOUNT_ID}}.dkr.ecr.{{AWS_REGION}}.amazonaws.com/{{PROJECT_NAME}}-frontend:latest
          ports:
            - containerPort: {{FRONTEND_PORT}}
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /
              port: {{FRONTEND_PORT}}
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: {{FRONTEND_PORT}}
            initialDelaySeconds: 15
            periodSeconds: 10
```

---

## infra/k8s/frontend/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{PROJECT_NAME}}-frontend
  namespace: {{PROJECT_NAME}}
spec:
  selector:
    app: {{PROJECT_NAME}}-frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: {{FRONTEND_PORT}}
  type: LoadBalancer
```

---

## .github/workflows/frontend.yml

```yaml
name: Frontend CI/CD

on:
  push:
    branches:
      - {{DEPLOY_BRANCH}}
    paths:
      - 'apps/frontend/**'
      - '.github/workflows/frontend.yml'

env:
  AWS_REGION: {{AWS_REGION}}
  ECR_REPOSITORY: {{PROJECT_NAME}}-frontend
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

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '{{NODE_VERSION}}'
          cache: 'npm'
          cache-dependency-path: apps/frontend/package-lock.json

      - name: Install dependencies
        working-directory: apps/frontend
        run: npm ci

      - name: Run tests
        working-directory: apps/frontend
        run: npm test --if-present

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
        working-directory: apps/frontend
        run: |
          IMAGE_URI=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ env.IMAGE_TAG }}
          docker build \
            {{#if API_URL_ENV}}--build-arg {{API_URL_ENV}}=${{ secrets.{{API_URL_ENV}} }} \{{/if}}
            -t $IMAGE_URI .
          docker push $IMAGE_URI
          echo "IMAGE_URI=$IMAGE_URI" >> $GITHUB_ENV

      - name: Update image tag in K8s manifest
        run: |
          sed -i "s|image: .*{{PROJECT_NAME}}-frontend:.*|image: ${{ env.IMAGE_URI }}|g" \
            infra/k8s/frontend/deployment.yaml

      - name: Commit and push updated manifest
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add infra/k8s/frontend/deployment.yaml
          git diff --staged --quiet || git commit -m "ci: update frontend image to ${{ env.IMAGE_TAG }}"
          git push
```
