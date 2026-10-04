# Sites

Cada arquivo `.caddy` aqui dentro é o site block de **um projeto**. O `Caddyfile`
principal importa todos eles automaticamente (`import sites/*.caddy`).

**Esta pasta começa vazia neste repositório** - nenhum arquivo de projeto é
versionado aqui (ver motivo na convenção abaixo). Isso significa que, logo
após clonar este repo, o Caddy não tem nenhum site configurado até que os
links simbólicos dos passos abaixo sejam criados. O Caddy sobe normalmente
mesmo assim (só loga um aviso tipo `No files matching import glob pattern` e
não serve nada) - não pule esse passo no primeiro deploy.

## Convenção: cada projeto é dono do seu arquivo

O arquivo de rotas de um projeto **não é mantido aqui**. Ele vive no próprio
repositório do projeto (ex.: `AgendaTEC/deploy/caddy/agendatec.caddy`) - assim,
quem mexe nas rotas do projeto não precisa abrir PR neste repositório de infra.

No deploy, **copie** o arquivo do projeto pra dentro desta pasta:

```bash
cp /caminho/para/AgendaTEC/deploy/caddy/agendatec.caddy sites/agendatec.caddy
```

Precisa repetir esse `cp` sempre que o projeto mudar a rota (ex.: um novo
endpoint de API). É um passo manual a mais, mas é intencional - ver "Por que
não link simbólico" abaixo.

Depois de adicionar/atualizar um arquivo aqui, recarregue o Caddy sem downtime:

```bash
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

## Adicionando um projeto novo

1. O projeto cria seu próprio `deploy/caddy/<projeto>.caddy` com o site block dele
   (domínio + rotas), e garante que os serviços que o Caddy precisa alcançar
   (ex.: o container web) estejam na rede externa `caddy_net` (ver README
   principal deste repositório).
2. No servidor, copie o arquivo para dentro de `sites/` (ver comando acima).
3. `docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`.

## Por que não link simbólico

Um link simbólico (`ln -s`) apontando pro repositório do projeto parece mais
conveniente (atualiza sozinho), mas não funciona só com a pasta `sites/`
montada no container - o Caddy não consegue seguir um link pra fora do que
está montado. A única forma de fazer o link funcionar seria montar no Caddy
um diretório mais amplo (ex. a pasta pai onde todos os repositórios ficam no
servidor), e isso dá ao Caddy acesso de leitura ao repositório **inteiro** de
cada projeto (`.env`, código-fonte, `.git`), não só ao arquivo `.caddy` que
ele precisa. Preferimos a cópia manual - repetitiva, mas mantém o Caddy
enxergando só o que ele realmente precisa.
