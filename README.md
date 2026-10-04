# labnext-caddy
Configuração centralizada do Caddy para gerenciamento de proxy reverso e HTTPS de múltiplos projetos via Docker Compose

## Por que esse repositório existe

O Caddy aqui é um **proxy reverso compartilhado**: um único Caddy, rodando
nesta máquina (servidor), expõe as portas 80/443 e distribui as requisições
para os containers de **vários projetos diferentes** - cada um rodando no seu
próprio `docker-compose.yml`, em repositórios separados.

Isso mantém os repositórios dos projetos (ex. `AgendaTEC`) livres do Caddy -
um dev que clona um projeto não precisa subir proxy reverso nem lidar com
certificado na própria máquina. Este repositório só é clonado e executado
**no servidor**.

## Estrutura

```
labnext-caddy/
├── docker-compose.yml   # só o serviço caddy
├── Caddyfile             # só importa sites/*.caddy
├── sites/
│   └── README.md         # convenção de como cada projeto adiciona seu site
│       (pasta começa vazia - ver sites/README.md antes do primeiro deploy)
└── logs/                 # logs de acesso por projeto (output dos site blocks)
```

## Pré-requisito: rede externa compartilhada

Os containers de cada projeto (o que precisa ser exposto publicamente, ex.:
o `web` do Django) precisam estar na mesma rede Docker que este Caddy. Essa
rede é externa e criada **uma única vez** no servidor, antes de subir
qualquer um dos dois lados:

```bash
docker network create caddy_net
```

Cada projeto declara essa rede como `external: true` (no seu `docker-compose.yml`,
ou melhor ainda, num override só de produção - veja como o AgendaTEC faz isso
em `docker-compose.prod.yml`, pra não exigir essa rede em dev) e anexa nela só
os serviços que precisam ser alcançados pelo Caddy (o banco de dados, redis
etc. continuam isolados na rede privada do próprio projeto).

## Subindo

```bash
docker network create caddy_net   # só na primeira vez, nunca mais precisa repetir
docker compose up -d
```

Isso sobe o Caddy, mas `sites/` começa vazio - sem nenhum projeto adicionado
(próxima seção), ele não serve nada.

## Adicionando um projeto novo

Ver [sites/README.md](sites/README.md).

## Recarregando depois de mudar um site

Não precisa reiniciar o container - o Caddy recarrega a config sem downtime:

```bash
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

## Certificados

Cada site block usa o domínio real do projeto (ex.:
`teste.agendatec.faculdadedombosco.net.br`) - o Caddy emite e renova o
certificado Let's Encrypt automaticamente para cada um, desde que o DNS do
domínio já aponte para o IP deste servidor e as portas 80/443 estejam
liberadas no firewall.

## Testando localmente

Dá pra validar a integração inteira (este repositório junto com o projeto
que ele serve) na sua própria máquina, sem DNS real e sem bater no Let's
Encrypt de verdade. A ideia é usar `localhost` como domínio só durante o
teste, já que o Caddy sabe gerar certificado interno sozinho pra esse nome
específico, sem precisar de ACME.

1. Crie a rede externa, se ainda não existir:
   ```bash
   docker network create caddy_net
   ```
2. No repositório do projeto (ex. `AgendaTEC`), copie o arquivo de rotas dele
   pra dentro de `sites/` (ver [sites/README.md](sites/README.md)):
   ```bash
   cp /caminho/para/AgendaTEC/deploy/caddy/agendatec.caddy sites/agendatec.caddy
   ```
3. Edite a cópia que você acabou de criar em `sites/agendatec.caddy` (não o
   arquivo original do projeto) e troque só a primeira linha, o domínio, pra
   `localhost`:
   ```
   localhost {
   ```
4. Suba o Caddy:
   ```bash
   docker compose up -d
   docker compose logs caddy --tail 20
   ```
   Confira que não aparece nenhum erro de `import` nos logs (indicaria que o
   arquivo em `sites/` não existe ou tem um problema de sintaxe).
5. No repositório do projeto, siga as instruções dele pra subir em modo
   produção (no caso do AgendaTEC, isso envolve descomentar
   `COMPOSE_FILE`/`COMPOSE_PROFILES` no `.env` dele e rodar
   `docker compose up -d`, ver o `README.md` daquele repositório).
6. Acesse `https://localhost` no navegador. Ele vai avisar que o certificado
   não é confiável (é o certificado interno do Caddy, gerado só pra esse
   teste), pode prosseguir mesmo assim.

Pra desfazer depois do teste:
- Apague ou substitua `sites/agendatec.caddy` por uma cópia nova do arquivo
  original do projeto (com o domínio de produção de volta).
- No repositório do projeto, reverta o que foi mudado no `.env` e derrube os
  containers com o mesmo profile usado pra subir (ex.:
  `docker compose --profile prod down`, não só `docker compose down` - veja
  o motivo no `README.md` do projeto).
