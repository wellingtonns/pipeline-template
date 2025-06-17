
# Documentação da Pipeline GitHub Actions: Build e Deploy do `apptest`

---

## Arquivo principal da pipeline

```yaml
name: Build and Deploy apptest

on:
  push:
    branches:
      - main

jobs:
  build:
    uses: wellingtonns/pipeline-template/.github/workflows/build-push.yml@main
    with:
      IMAGE_NAME: welignton/apptest
      APP_NAME: apptest
    secrets: inherit

  sync-argocd:
    needs: build
    uses: wellingtonns/pipeline-template/.github/workflows/argocd-sync.yml@main
    with:
      APP_NAME: apptest
    secrets: inherit
```

### Explicação

* Dispara a pipeline a cada push na branch `main`.
* Job `build`: executa o template para construir e publicar a imagem Docker.
* Job `sync-argocd`: espera o job `build` completar e então executa o template para sincronizar o ArgoCD.
* Usa `secrets: inherit` para usar os segredos do repositório.

---

## Template: Build e Push da imagem Docker (`build-push.yml`)

```yaml
name: Build and Push Image

on:
  workflow_call:
    inputs:
      IMAGE_NAME:
        required: true
        type: string
      APP_NAME:
        required: true
        type: string
    secrets:
      DOCKERHUB_USERNAME:
        required: true
      DOCKERHUB_TOKEN:
        required: true

jobs:
  build-and-push:
    name: Build and Push Image
    runs-on: ubuntu-latest

    env:
      IMAGE_NAME: ${{ inputs.IMAGE_NAME }}
      APP_NAME: ${{ inputs.APP_NAME }}

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Login no Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Definir TAG
        run: echo "TAG=${GITHUB_SHA::7}" >> $GITHUB_ENV

      - name: Build da imagem Docker
        run: |
          docker build -t $IMAGE_NAME:${TAG} .
          docker tag $IMAGE_NAME:${TAG} $IMAGE_NAME:latest

      - name: Push da imagem Docker
        run: |
          docker push $IMAGE_NAME:${TAG}
          docker push $IMAGE_NAME:latest

      - name: Atualizar deployment.yaml
        run: |
          sed -i "s|image: $IMAGE_NAME.*|image: $IMAGE_NAME:${TAG}|g" manifestos/deployment.yaml

      - name: Commit e Push do Manifesto
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git remote set-url origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/${{ github.repository }}
          git add manifestos/deployment.yaml
          git commit -m "Atualiza imagem para tag ${TAG}" || echo "Nenhuma alteração para commit"
          git push origin HEAD
```

### Explicação passo a passo

1. **Checkout:** Clona o código fonte do repositório para o runner.

2. **Login no Docker Hub:** Autentica no Docker Hub usando as credenciais fornecidas nos segredos.

3. **Definir TAG:** Define a variável `TAG` com os 7 primeiros caracteres do SHA do commit atual. Isso garante que cada imagem tenha uma tag única.

4. **Build da imagem Docker:** Constrói a imagem Docker localmente com a tag definida e também marca com a tag `latest`.

5. **Push da imagem Docker:** Envia as duas tags para o Docker Hub.

6. **Atualizar deployment.yaml:** Usa o `sed` para substituir a linha de imagem Docker no manifesto Kubernetes (`deployment.yaml`) com a nova tag da imagem.

7. **Commit e push do manifesto:** Configura o git, adiciona o manifesto modificado e faz commit + push para o branch atual. Se não houver mudanças, não gera erro.

---

## Template: Sincronização com ArgoCD (`argocd-sync.yml`)

```yaml
name: Sync ArgoCD

on:
  workflow_call:
    inputs:
      APP_NAME:
        required: true
        type: string
    secrets:
      ARGOCD_URL:
        required: true
      ARGOCD_USER:
        required: true
      ARGOCD_PASS:
        required: true

jobs:
  argocd-sync:
    name: Sync ArgoCD
    runs-on: ubuntu-latest

    env:
      APP_NAME: ${{ inputs.APP_NAME }}
      ARGOCD_URL: ${{ secrets.ARGOCD_URL }}
      ARGOCD_USER: ${{ secrets.ARGOCD_USER }}
      ARGOCD_PASS: ${{ secrets.ARGOCD_PASS }}

    steps:
      - name: Instalar jq
        run: |
          sudo apt-get update
          sudo apt-get install -y jq

      - name: Login no ArgoCD
        id: login
        run: |
          echo "🔐 Fazendo login no ArgoCD"
          TOKEN=$(curl -sk -X POST -H "Content-Type: application/json" \
            -d '{"username":"'"$ARGOCD_USER"'","password":"'"$ARGOCD_PASS"'"}' \
            https://$ARGOCD_URL/api/v1/session | jq -r .token)

          if [[ -z "$TOKEN" || "$TOKEN" == "null" ]]; then
            echo "❌ Erro no login no ArgoCD"
            exit 1
          fi

          echo "ARGOCD_TOKEN=$TOKEN" >> $GITHUB_ENV

      - name: Verificar Status no ArgoCD
        run: |
          echo "🔍 Verificando status do app $APP_NAME"

          degraded_found=false

          for i in {1..8}
          do
            RESPONSE=$(curl -sk -H "Authorization: Bearer $ARGOCD_TOKEN" https://$ARGOCD_URL/api/v1/applications/$APP_NAME)
            STATUS=$(echo "$RESPONSE" | jq -r '.status.sync.status')
            HEALTH=$(echo "$RESPONSE" | jq -r '.status.health.status')

            if [[ "$STATUS" == "Synced" && "$HEALTH" == "Healthy" ]]; then
              echo "Aplicação sincronizada e saudável ✅"
              exit 0
            fi

            if [[ "$HEALTH" == "Degraded" ]]; then
              degraded_found=true
            fi

            echo "Status atual: $STATUS / $HEALTH. Tentativa $i/8 - Aguardando 30s..."
            sleep 30
          done

          if [[ "$degraded_found" == "true" ]]; then
            echo "Aplicação com status Degraded ❌"
            exit 1
          else
            echo "Aplicação não sincronizada após 8 tentativas ❌"
            exit 1
          fi
```

### Explicação passo a passo

1. **Instalar `jq`:** Ferramenta para processar JSON no shell.

2. **Login no ArgoCD:** Autentica na API do ArgoCD via POST para obter token JWT.

3. **Verificar status da aplicação:**

   * Em até 8 tentativas com intervalo de 30 segundos cada, faz GET para o endpoint de status da aplicação.
   * Verifica os campos `.status.sync.status` e `.status.health.status`.
   * Se estiver `Synced` e `Healthy`, a pipeline finaliza com sucesso.
   * Se detectar qualquer status `Degraded`, marca como falha.
   * Caso não atinja sucesso após as tentativas, encerra com erro.

---

# Resumo do fluxo completo

1. O push na branch `main` dispara a pipeline.
2. O job `build` constrói a imagem Docker, faz push para o Docker Hub, atualiza o manifesto Kubernetes e faz commit.
3. O job `sync-argocd` sincroniza o ArgoCD, monitorando a implantação e garantindo que a aplicação esteja saudável.

---