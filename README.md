# Ambiente Docker para aplicações PHP

Este Compose oferece ambientes separados para produção e desenvolvimento, com
Nginx e PHP-FPM em containers distintos. Inclui MariaDB, phpMyAdmin, coleta de
logs com Fluent Bit e Loki e métricas dos containers via API Docker, Prometheus
e dashboards provisionados no Grafana.

## Requisitos

- Docker Engine e Docker Compose v2.
- Um host Linux com o driver de logs Docker `json-file` (padrão), para a coleta
  de logs em `/var/lib/docker/containers`.

## Preparar e iniciar

1. Copie `.env.example` para `.env` e altere as senhas de MariaDB e Grafana.
2. Coloque o código-fonte da aplicação de produção em `apps/producao/` e o de
   desenvolvimento em `apps/desenvolvimento/`.
3. Por padrão, as duas aplicações usam `/var/www/html/public` como raiz web,
   apropriado para Laravel. Para uma aplicação PHP cuja raiz seja a própria
   pasta do projeto, altere `PRODUCTION_DOCUMENT_ROOT` e/ou
   `DEVELOPMENT_DOCUMENT_ROOT` em `.env` para `/var/www/html`.
4. Escolha os componentes no `COMPOSE_PROFILES` do `.env` e execute:

   ```sh
   docker compose up -d --build
   ```

O `.env.example` ativa todos os perfis por padrão. Os perfis disponíveis são
`production`, `development`, `database` e `observability`. Remova do
`COMPOSE_PROFILES` os componentes que não deseja iniciar. Por exemplo, para
produção com banco e observabilidade, use:

```env
COMPOSE_PROFILES=production,database,observability
```

Produção e desenvolvimento podem ser iniciados sem o perfil `database`, mas
aplicações que dependem do banco precisam dele ativo. O perfil `database`
inclui MariaDB e phpMyAdmin.

Para desligar componentes já iniciados sem apagar volumes, execute `docker
compose stop` com os serviços correspondentes, por exemplo:

```sh
docker compose --profile development stop php-development nginx-development
docker compose --profile observability stop grafana prometheus docker-metrics fluent-bit loki
docker compose --profile database stop phpmyadmin mysql
```

Para habilitá-los novamente, ajuste `COMPOSE_PROFILES` no `.env` e rode
`docker compose up -d`. Os dados persistentes ficam nos volumes Docker.

Os containers PHP incluem Composer e as extensões `bcmath`, `gd`, `intl`,
`pdo_mysql`, `opcache` e `zip`. Para instalar dependências Laravel, por exemplo:

```sh
docker compose exec php-production composer install --no-dev --optimize-autoloader
docker compose exec php-development composer install
```

Configure cada aplicação PHP para acessar o banco usando `mysql` como host,
`3306` como porta e as credenciais definidas em `.env`. O banco inicial
`MYSQL_DATABASE` também é criado pelo MariaDB ao inicializar o volume pela primeira
vez. Para Laravel, rode migrações no ambiente apropriado, por exemplo:

```sh
docker compose exec php-production php artisan migrate --force
```

## Endereços

| Serviço | Endereço padrão |
| --- | --- |
| Aplicação de produção | `http://localhost:8000` |
| Aplicação de desenvolvimento | `http://localhost:8001` |
| phpMyAdmin | `http://localhost:8002` |
| Grafana | `http://localhost:3000` |
| MariaDB | `127.0.0.1:3306` |

No phpMyAdmin, entre com o usuário e a senha `MYSQL_USER` e `MYSQL_PASSWORD`
configurados em `.env`. O Grafana é provisionado com Loki e Prometheus como
datasources, além do dashboard **Docker - Containers** na pasta **Docker**.
Ele apresenta CPU, memória e tráfego de rede por container e permite filtrar
qual container exibir. Telegraf consulta a API Docker a cada 15 segundos e
Prometheus mantém as métricas por 15 dias no seu volume. Para consultar logs,
use **Explore**, selecione Loki e execute `{job="docker"}`; a retenção
configurada no Loki é de sete dias.

## Configuração e dados

As portas e raízes web podem ser alteradas no `.env`. O MariaDB fica vinculado a
`127.0.0.1` por padrão, evitando expor o banco à rede. Para acesso remoto,
configure `MYSQL_BIND_ADDRESS` conscientemente e restrinja o acesso com firewall.

Os dados do MariaDB, Loki e Grafana persistem em volumes Docker. Faça backup desses
volumes antes de manutenção. O Compose configura rotação dos logs do Docker em
arquivos de até 10 MB, mantendo três arquivos por container; Fluent Bit envia
esses logs ao Loki. O coletor de métricas acessa o socket do Docker, que
concede controle administrativo sobre o daemon; mantenha esse acesso restrito
e habilite observabilidade apenas quando necessário.

> **Atenção na troca de MySQL:** o volume `mysql-data` não deve ser reutilizado
> diretamente com MariaDB como se fosse uma migração. Faça backup e migre os
> dados pelo procedimento apropriado antes de trocar a imagem em uma instalação
> que já tenha um banco inicializado.

Este é um ponto de partida para hospedar aplicações existentes, não uma
configuração completa de produção: não inclui HTTPS, proxy reverso, políticas de
backup automatizadas nem endurecimento específico da aplicação. Planeje TLS,
firewall, atualização de imagens, permissões dos arquivos e backup no ambiente
do cliente antes de expor os serviços à Internet.
