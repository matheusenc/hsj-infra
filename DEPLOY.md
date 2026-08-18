# Deploy — VPS Oracle

Infraestrutura do Hospital São José. Três repositórios:

| Repositório | Conteúdo | Publica |
|---|---|---|
| `hsj-back-end` | API .NET 10 + `Dockerfile` | `ghcr.io/<voce>/hsj-api` |
| `hsj-front-end` | Angular 22 + `Dockerfile` | `ghcr.io/<voce>/hsj-web` |
| `hsj-infra` (este) | `docker-compose.yml`, `Caddyfile`, `.env` | — |

**Na VPS existe apenas o clone deste repositório.** As imagens são construídas
pelo GitHub Actions a cada push na `main`; a VPS só as baixa. A máquina tem
1 GB de RAM — compilar Angular e .NET nela seria lento e disputaria memória com
o que já está no ar.

Domínios:

| Domínio | Destino |
|---|---|
| `hsjnovaera.com.br` | site novo (container `web`) |
| `www.hsjnovaera.com.br` | redireciona para o domínio sem www |
| `api.hsjnovaera.com.br` | API (container `api`) |

---

## Estado atual da VPS

Já feito em `137.131.248.218` (Ubuntu 24.04, 2 vCPU, 954 MB de RAM, 45 GB):

- Docker 29.7.2 instalado e habilitado no boot;
- swap de 2 GB criado e registrado no `/etc/fstab`;
- o Postgres de teste foi removido junto do volume dele.

Falta o que está descrito abaixo.

---

## 1. Painel da Oracle Cloud — portas

**Networking → Virtual Cloud Networks → sua VCN → Security Lists → Default.**

Adicionar duas regras de ingresso:

| Source CIDR | Protocolo | Porta |
|---|---|---|
| `0.0.0.0/0` | TCP | `80` |
| `0.0.0.0/0` | TCP | `443` |

**Remover a regra que libera a 5432.** Ela existe hoje e deixava o Postgres de
teste acessível pela internet — confirmado por teste externo. No desenho novo o
banco não publica porta nenhuma, então a regra só representa risco.

> Se a instância usa Network Security Group em vez de Security List, as mesmas
> mudanças vão no NSG.

---

## 2. Cloudflare — registros DNS

Hoje o Cloudflare aponta para a Hostinger, onde o site legado está no ar.
**Nada aqui derruba o site atual**: só o subdomínio `api` sai do Hostinger
agora. O domínio raiz migra no passo 6, quando o site novo estiver validado.

DNS → Records → Add record:

| Type | Name | IPv4 | Proxy status | TTL |
|---|---|---|---|---|
| `A` | `api` | `137.131.248.218` | **DNS only** (nuvem cinza) | Auto |

Confira a propagação antes de seguir, do seu Windows:

```powershell
nslookup api.hsjnovaera.com.br 1.1.1.1
```

Precisa responder o IP da VPS. Se responder `104.x` ou `172.67.x`, o proxy
ficou ligado — volte e mude para DNS only.

> **Por que DNS only:** com a nuvem laranja ligada, o desafio da Let's Encrypt
> não chega na VPS e o Caddy não emite o certificado. Para uma API o proxy
> também não traz ganho — cache de CDN não ajuda em resposta autenticada. Se
> ainda assim quiser ligá-lo depois, veja o §8.

---

## 3. Publicar as imagens

Nos dois repositórios de aplicação, faça o push da `main`. O workflow
`.github/workflows/publicar-imagem.yml` roda sozinho e publica no GHCR.

Depois, em cada repositório no GitHub: **Packages → o pacote → Package settings
→ Change visibility → Public.** Com os pacotes públicos a VPS baixa sem
autenticação, o que evita guardar um token nela.

Se preferir mantê-los privados, será preciso um Personal Access Token com
escopo `read:packages` e, na VPS:

```bash
echo "SEU_TOKEN" | docker login ghcr.io -u SEU_USUARIO --password-stdin
```

---

## 4. VPS — configurar

```bash
ssh -i ~/.ssh/hsj.key ubuntu@137.131.248.218

mkdir -p ~/apps && cd ~/apps
git clone https://github.com/SEU_USUARIO/hsj-infra.git
cd hsj-infra

cp .env.example .env
```

Gere os segredos:

```bash
openssl rand -base64 24    # POSTGRES_PASSWORD
openssl rand -base64 48    # JWT_SIGNING_KEY
```

Preencha `GHCR_OWNER`, as duas chaves acima, `SEED_ADMIN_PASSWORD` e
`ACME_EMAIL`:

```bash
nano .env
chmod 600 .env
```

Firewall do host — o Docker publica portas por um caminho próprio no iptables,
então quem realmente controla 80/443 é a Security List do §1; o ufw aqui
protege o SSH e o que rodar fora de container:

```bash
sudo apt update && sudo apt install -y ufw fail2ban
sudo ufw allow OpenSSH
sudo ufw allow 80,443/tcp
sudo ufw --force enable
sudo systemctl enable --now fail2ban
```

---

## 5. Primeira subida

```bash
cd ~/apps/hsj-infra
docker compose pull
docker compose up -d
docker compose logs -f
```

O que esperar, nesta ordem:

1. `postgres` → `database system is ready to accept connections`
2. `api` → a lista de migrations do FluentMigrator e depois
   `Now listening on: http://[::]:8080`
3. `caddy` → `certificate obtained successfully` para `api.hsjnovaera.com.br`

`Ctrl+C` sai só do log; os containers seguem rodando.

### Verificação

```bash
docker compose ps                        # os quatro em running/healthy
curl -I https://api.hsjnovaera.com.br    # do seu Windows também
```

`404` na raiz é resultado **correto** — a API não expõe nada em `/`. O que
importa é o certificado ser aceito sem aviso. Teste um endpoint real:

```bash
curl -i -X POST https://api.hsjnovaera.com.br/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@hsjnovaera.com.br","password":"SUA_SENHA_DO_SEED"}'
```

Deve voltar `200` com um token.

---

## 6. Virar o site novo

Só depois de validar a API e conferir o site novo. Enquanto isso o
`hsjnovaera.com.br` continua servido pela Hostinger.

Para conferir antes de mexer no DNS, adicione ao `hosts` do seu Windows
(`C:\Windows\System32\drivers\etc\hosts`, como administrador):

```
137.131.248.218 hsjnovaera.com.br
```

O certificado vai acusar erro nesse teste — a Let's Encrypt ainda não validou o
domínio —, mas dá para navegar o site inteiro. Remova a linha depois.

Quando estiver satisfeito:

1. Cloudflare → registro `A` de `hsjnovaera.com.br` e de `www` apontando para
   `137.131.248.218`, ambos em **DNS only**.
2. Aguarde a propagação e acesse `https://hsjnovaera.com.br`. O Caddy emite os
   certificados no primeiro acesso.
3. Só então cancele a hospedagem da Hostinger.

---

## 7. Rotina

### Deploy de uma alteração

Faça o push na `main` do repositório correspondente, aguarde o Actions terminar
e, na VPS:

```bash
cd ~/apps/hsj-infra
docker compose pull
docker compose up -d
```

As migrations rodam sozinhas no start da API — não há passo separado de banco.

### Voltar atrás

Pegue o sha curto do build anterior em
`github.com/<voce>/<repo>/pkgs/container` e aponte no `.env`:

```bash
nano .env        # API_TAG=sha-1a2b3c4
docker compose up -d
```

### Comandos do dia a dia

```bash
docker compose logs -f api      # log da API
docker compose restart api      # reiniciar só a API
docker compose down             # derrubar tudo (volumes preservados)
docker system prune -af         # limpar imagens antigas quando o disco encher
```

### Backup

Os dados vivem em dois volumes: `hsj_postgres_data` e `hsj_documents_data`.
Perder qualquer um é perda real — o segundo guarda os PDFs de transparência.

```bash
mkdir -p ~/backups
cd ~/apps/hsj-infra

docker compose exec -T postgres pg_dump -U hsj hsjdb | gzip > ~/backups/db-$(date +%F).sql.gz

docker run --rm -v hsj_documents_data:/data -v ~/backups:/backup alpine \
  tar czf /backup/docs-$(date +%F).tar.gz -C /data .
```

Diariamente às 3h, via `crontab -e` (o `%` precisa de escape no cron):

```
0 3 * * * cd ~/apps/hsj-infra && docker compose exec -T postgres pg_dump -U hsj hsjdb | gzip > ~/backups/db-$(date +\%F).sql.gz
```

Baixe as cópias para fora da VPS periodicamente — backup que só existe no mesmo
host não é backup.

---

## 8. Opcional — ligar o proxy do Cloudflare

Só depois dos certificados já emitidos.

1. Cloudflare → **SSL/TLS → Overview** → modo **Full (strict)**.
   Nunca `Flexible`: ele fala HTTP com a VPS, o Caddy responde com redirect e
   o navegador entra em loop.
2. DNS → mudar o Proxy status para **Proxied**.

O plano gratuito limita upload a 100 MB, acima dos 26 MB da API — sem conflito.
Se a renovação do certificado falhar mais para frente, volte para DNS only,
deixe o Caddy renovar e religue.

---

## Problemas comuns

| Sintoma | Causa provável |
|---|---|
| `curl` de fora não responde, mas de dentro da VPS funciona | Ingress rule faltando na Security List (§1) |
| Caddy repete `could not get certificate` | Registro ainda em Proxied, ou DNS não propagou (§2) |
| `docker compose pull` dá `denied` | Pacote privado no GHCR — torne-o público ou faça `docker login` (§3) |
| `api` reinicia em loop na primeira subida | `.env` incompleto — veja `docker compose logs api` |
| Erro de CORS no site | `CORS_ORIGIN_SITE` com barra no final ou com `http://` em vez de `https://` |
| Redirect infinito no navegador | SSL/TLS do Cloudflare em `Flexible` (§8) |
