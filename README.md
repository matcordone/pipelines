# 🚀 Build & Push Image to DEV ECR (GitHub Actions)

Este repositorio contiene el workflow encargado de **construir y publicar imágenes Docker en el ECR de DEV** cada vez que se hace un push al branch `dev`.  
El objetivo es tener un flujo rápido, seguro y totalmente automatizado sin necesidad de credenciales de larga duración.

---

## 📌 Cómo funciona el pipeline

Cada vez que hacés:

### git push origin dev


GitHub Actions dispara automáticamente este workflow y ejecuta:

1. **Clonado del repo**  
2. **Asunción de un rol IAM en la cuenta DEV vía OIDC**  
   - Sin Access Keys  
   - Autenticación segura y temporal  
3. **Login contra Amazon ECR (DEV)**  
4. **Build de la imagen Docker**  
5. **Push de la imagen usando el commit SHA como tag**  
6. **Fin del job**

El resultado final es una imagen inmutable publicada en tu repositorio ECR dev.

---

## 📁 Archivo del workflow

El pipeline vive en:



.github/workflows/dev-ecr.yml


Y solo se ejecuta cuando el trigger es:

```yaml
on:
  push:
    branches:
      - dev

🧱 Dockerfile esperado

Este workflow asume que existe:

./Dockerfile


Si usás otro path, simplemente ajustá la variable DOCKERFILE_PATH dentro del workflow.

🧩 Tagging de imágenes

El workflow taggea las imágenes así:

<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<REPO_NAME>:<COMMIT_SHA>

```

## Ventajas:

Imágenes inmutables

Rollbacks triviales

Trazabilidad precisa

### 🔐 Autenticación vía OpenID Connect (OIDC)

Este pipeline no usa Access Keys.
GitHub accede a AWS asumiendo un rol vía OIDC:

arn:aws:iam::<ACCOUNT_DEV>:role/<ROLE_NAME>




## Esto aporta:

```
Autenticación sin credenciales persistentes

Menor superficie de ataque

Credenciales efímeras

Seguridad basada en identidad del repo + branch

🛠️ Configuración necesaria en AWS (Cuenta DEV)
1. Trust Policy del rol (permitir OIDC desde GitHub)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_DEV>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<TU_ORG>/<TU_REPO>:ref:refs/heads/dev"
        }
      }
    }
  ]
}

2. Políticas necesarias para push a ECR o Rol ya condigurado ECR PowerUser

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage"
      ],
      "Resource": "*"
    }
  ]
}

```

🟦 Variables y Secrets requeridos en GitHub
Variables (Settings → Variables → Actions)
Nombre	Descripción
ECR_REPOSITORY	Nombre del repo en ECR (ej: my-app-dev)
Secrets (Settings → Secrets → Actions)
Nombre	Descripción
AWS_ACCOUNT_DEV	ID de la cuenta AWS DEV

🧪 Dockerfile de ejemplo simple
FROM alpine:3.20
CMD ["echo", "Hello from DEV build!"]

🛠️ Resumen del workflow (dev-ecr.yml)
name: Build & Push Image to DEV ECR

on:
  push:
    branches:
      - dev

env:
  AWS_REGION: "us-east-1"
  ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
  DOCKERFILE_PATH: "./Dockerfile"

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_DEV }}:role/GitHubActionsPushToECRRole

      - name: Login to Amazon ECR (DEV)
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker Image
        run: |
          SHA="${{ github.sha }}"
          REGISTRY="${{ steps.ecr.outputs.registry }}"
          IMAGE_URI="$REGISTRY/${{ env.ECR_REPOSITORY }}"

          echo "Building image: $IMAGE_URI:$SHA"

          docker build \
            -f "${{ env.DOCKERFILE_PATH }}" \
            -t "$IMAGE_URI:$SHA" .

      - name: Push Image
        run: |
          SHA="${{ github.sha }}"
          REGISTRY="${{ steps.ecr.outputs.registry }}"
          IMAGE_URI="$REGISTRY/${{ env.ECR_REPOSITORY }}"

          docker push "$IMAGE_URI:$SHA"

      - name: Done
        run: echo "Image pushed successfully."