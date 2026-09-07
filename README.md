# OCS Inventory + osTicket Lab Template

Laboratório virtual para validação de uma plataforma integrada de inventário de ativos e service desk usando Oracle VirtualBox, osTicket e OCS Inventory NG.

> **Status:** validado em 07/09/2026  
> **Tipo:** laboratório / template de recuperação  
> **Hypervisor:** Oracle VirtualBox

## Objetivo

Este projeto documenta a construção, validação e preservação de uma máquina virtual de laboratório contendo:

- osTicket para service desk;
- OCS Inventory NG para inventário de hardware e software;
- Apache/httpd, PHP-FPM e MariaDB;
- agente OCS Inventory para Windows;
- backup lógico do banco e snapshot do VirtualBox.

O ambiente foi validado antes de qualquer utilização em produção.

## Arquitetura

```text
Windows endpoint
    |
    | HTTP POST /ocsinventory
    v
Linux VM
    |
    +-- Apache/httpd
    +-- OCS Inventory NG Reports
    +-- PHP/PHP-FPM
    +-- MariaDB: ocsweb
    +-- osTicket
```

## Endereços do laboratório

Substitua os placeholders conforme o ambiente local. Não publique endereços internos reais se o repositório for público.

```text
OCS Reports:        http://<LAB_SERVER_IP>/ocsreports/
OCS communication: http://<LAB_SERVER_IP>/ocsinventory
osTicket:            http://<LAB_SERVER_IP>/<OSTICKET_PATH>/
MariaDB database:    ocsweb
MariaDB user:        ocs
```

## Componentes validados

| Componente | Estado |
|---|---|
| OCS Reports | Operacional |
| OCS communication endpoint | Operacional |
| osTicket | Operacional |
| Apache/httpd | Ativo e habilitado no boot |
| PHP-FPM | Ativo e habilitado no boot |
| MariaDB | Ativo e habilitado no boot |
| `install.php` do OCS | Removido; HTTP 404 |
| Windows Agent | 2.11.0.1 validado |
| Inventário Windows | Hardware e software recebidos |
| Snapshot VirtualBox | Criado |

## Validação realizada

O agente OCS Inventory NG Agent 2.11.0.1 foi instalado em um notebook Windows 11 Pro e configurado com:

```text
Communication server: http://<LAB_SERVER_IP>/ocsinventory
SSL: disabled for the HTTP laboratory
Tag: LAB-OCS
Mode: Windows Service
```

Inventário recebido no banco:

```text
Hostname: test
User: default
Operating system: Microsoft Windows 11 Pro
```

A consulta usada para confirmar o recebimento foi:

```sql
SELECT ID, NAME, USERID, OSNAME, LASTDATE
FROM hardware
ORDER BY LASTDATE DESC
LIMIT 10;
```

A console exibiu hardware, software e demais informações do endpoint.

## Checks de validação

Execute na VM Linux:

```bash
systemctl is-active httpd php-fpm mariadb
systemctl is-enabled httpd php-fpm mariadb

curl -sS -o /dev/null -w 'ocsreports: HTTP %{http_code}\n' \\
  http://localhost/ocsreports/

curl -sS -o /dev/null -w 'install.php: HTTP %{http_code}\n' \\
  http://localhost/ocsreports/install.php

mysql -u ocs -p ocsweb -e "
SELECT ID, NAME, USERID, OSNAME, LASTDATE
FROM hardware
ORDER BY LASTDATE DESC
LIMIT 10;
"
```

Resultados esperados:

```text
ocsreports: HTTP 200
install.php: HTTP 404
```

O endpoint `/ocsinventory` não deve ser validado abrindo-o no navegador. Ele espera uma requisição de inventário enviada pelo agente; um `GET` direto pode retornar HTTP 400.

## Backup do banco

Criar backup lógico sem incluir senha no comando:

```bash
mkdir -p /root/backups/ocs

mysqldump \\
  --single-transaction \\
  --routines \\
  --triggers \\
  --events \\
  -u ocs -p ocsweb \\
  > /root/backups/ocs/ocsweb-$(date +%F-%H%M%S).sql

gzip /root/backups/ocs/ocsweb-*.sql
ls -lh /root/backups/ocs/
```

O arquivo SQL compactado deve ser armazenado fora do GitHub se contiver inventários, usuários ou dados internos.

## Instalação do agente Windows

### Instalação interativa

No instalador do OCS Windows Agent:

- Server URL: `http://<LAB_SERVER_IP>/ocsinventory`;
- usuário e senha do servidor: vazios, salvo configuração explícita de autenticação;
- validação de certificado: desabilitada no laboratório HTTP;
- instalação como Windows Service;
- verbose log habilitado para testes;
- coleta de softwares habilitada;
- tag: `LAB-OCS`;
- inventário imediato habilitado.

### Instalação silenciosa

Ajuste o nome do instalador e o endereço do servidor:

```cmd
OCS-Windows-Agent-2.11.0.1_x64.exe /S /SERVER=http://<LAB_SERVER_IP>/ocsinventory /SSL=0 /TAG=LAB-OCS /DEBUG=2 /NOW /NOSPLASH /NP
```

Validar no Windows:

```cmd
sc query "OCS Inventory Service"
type "C:\ProgramData\OCS Inventory NG\Agent\ocsinventory.ini"
```

O arquivo deve conter um servidor equivalente a:

```ini
[HTTP]
Server=http://<LAB_SERVER_IP>/ocsinventory
SSL=0
```

## Snapshot do VirtualBox

Snapshot validado criado em 07/09/2026:

```text
VALIDADO - osTicket + OCS Inventory - 2026-09-07
```

O snapshot representa o ponto em que:

- osTicket estava operacional;
- OCS Reports estava acessível;
- o instalador web do OCS havia sido removido;
- o endpoint Windows havia enviado inventário;
- hardware e software haviam sido confirmados na console;
- o backup lógico do banco já havia sido criado.

## Segurança e publicação

Não versionar:

- senhas;
- dumps SQL reais;
- chaves privadas;
- certificados com chave privada;
- arquivos de configuração com credenciais;
- inventário de endpoints reais;
- nomes de domínio e IPs internos, caso o repositório seja público;
- arquivos `.env`, históricos de shell ou backups pessoais.

Usar placeholders como:

```text
<LAB_SERVER_IP>
<OCS_FQDN>
<MYSQL_PASSWORD>
<DOMAIN>
<OSTICKET_PATH>
```

Antes do commit:

```bash
git status
git diff --cached
grep -RniE 'password|passwd|secret|token|private_key|***\.***\.|***\.***\.' . --exclude-dir=.git
```

Revise falsos positivos manualmente, mas não ignore nenhuma ocorrência sem confirmar que ela não contém segredo ou dado pessoal.

## Estrutura recomendada

```text
ocsinventory-osticket-lab-template/
├── README.md
├── LICENSE
├── .gitignore
├── docs/
│   ├── 01-architecture.md
│   ├── 02-prerequisites.md
│   ├── 03-osticket-validation.md
│   ├── 04-ocsinventory-installation.md
│   ├── 05-windows-agent.md
│   ├── 06-validation-checklist.md
│   ├── 07-backup-snapshot-restore.md
│   └── 08-security-hardening.md
├── scripts/
│   ├── linux/
│   │   ├── backup-ocs-db.sh
│   │   └── validate-ocs.sh
│   └── windows/
│       ├── install-ocs-agent-lab.cmd
│       └── install-ocs-agent-production.example.cmd
├── config-examples/
│   ├── z-ocsinventory-server.conf.example
│   └── ocsinventory.ini.example
└── screenshots/
    └── README.md
```

## Roadmap

- [ ] Revisar e adaptar a documentação original do projeto osTicket.
- [ ] Adicionar procedimento detalhado de instalação do osTicket.
- [ ] Adicionar procedimento detalhado de instalação do OCS Inventory.
- [ ] Criar scripts Linux de validação e backup.
- [ ] Criar pacote Windows de instalação do agente com placeholders.
- [ ] Testar restauração do dump SQL.
- [ ] Criar clone higienizado da VM.
- [ ] Avaliar HTTPS com certificado válido.
- [ ] Testar tags, grupos e deployment.
- [ ] Revisar todo o conteúdo antes de tornar o repositório público.
- [ ] Preparar publicação técnica no LinkedIn.

## Licença

Escolha uma licença compatível com os arquivos e scripts que serão publicados. Para documentação e scripts próprios, MIT ou Apache-2.0 podem ser opções adequadas; confirme as licenças dos componentes de terceiros antes de redistribuir arquivos deles.
