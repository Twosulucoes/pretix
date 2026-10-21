# pretix em produção com Docker Compose

Esta pasta sobe o pretix deste repositório (com a tradução pt-BR) junto com
PostgreSQL, Redis, um processo de tarefas periódicas e, em produção, o Caddy,
que cuida do HTTPS automaticamente.

## O que você precisa

- Um servidor Linux (Ubuntu 24.04 ou Debian 12/13) com pelo menos 2 vCPU, 4 GB de RAM e 40 GB de disco.
- Docker e o plugin Docker Compose instalados (`curl -fsSL https://get.docker.com | sh`).
- Um domínio ou subdomínio (por exemplo `ingressos.seudominio.com.br`) com um registro DNS do tipo A apontando para o IP do servidor.
- Uma conta de SMTP para enviar e-mails (Amazon SES, Brevo, Mailgun, Zoho, o SMTP do seu provedor etc.).
- As portas 80 e 443 liberadas no firewall do servidor.

## Testar no seu computador

```bash
git clone https://github.com/Twosulucoes/pretix.git
cd pretix/deployment/compose
cp .env.example .env
# No .env, troque PRETIX_PRETIX_URL para http://localhost:8000
docker compose up -d --build
```

A primeira construção da imagem demora (instala Python, Node e compila os arquivos estáticos).
Depois abra http://localhost:8000/control/ e entre com `admin@localhost` e a senha `admin`.
Troque essa senha logo no primeiro acesso.

## Colocar em produção

```bash
cd pretix/deployment/compose
cp .env.example .env
nano .env          # preencha domínio, senha do banco e SMTP
docker compose --profile producao up -d --build
```

O Caddy emite o certificado HTTPS sozinho na primeira visita ao domínio.
O sistema fica em `https://SEU-DOMINIO/control/`.

## Comandos do dia a dia

| O que fazer | Comando |
| --- | --- |
| Ver os logs | `docker compose logs -f pretix` |
| Reiniciar | `docker compose restart pretix` |
| Parar tudo | `docker compose --profile producao down` |
| Atualizar depois de um `git pull` | `docker compose --profile producao up -d --build` |
| Abrir o shell do Django | `docker compose exec pretix pretix shell` |

## Backup

Faça backup diário do banco e da pasta de dados (ingressos, faturas, imagens):

```bash
docker compose exec -T postgres pg_dump -U pretix pretix | gzip > backup-$(date +%F).sql.gz
docker run --rm -v compose_pretix_data:/data -v "$PWD":/backup debian tar czf /backup/dados-$(date +%F).tar.gz -C /data .
```

Guarde os arquivos fora do servidor (outro provedor, S3, Google Drive).
O nome do volume (`compose_pretix_data`) segue o nome da pasta; confira com `docker volume ls`.

## Observações

- O pt-BR é marcado como "incubating" no pretix e fica escondido em produção.
  O `.env.example` já libera com `PRETIX_LANGUAGES_ALLOW_INCUBATING=pt-br`.
- Qualquer opção do `pretix.cfg` pode ser definida no `.env` no formato
  `PRETIX_<SEÇÃO>_<OPÇÃO>`, por exemplo `PRETIX_MAIL_HOST` para `[mail] host`.
- Pix e boleto não existem nos plugins de pagamento nativos; Stripe e PayPal funcionam para cartão.
