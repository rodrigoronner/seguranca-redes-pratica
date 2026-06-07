# Segurança de Redes na Prática

**Análise de Tráfego, Varredura, Firewall e Políticas de Acesso**

Repositório oficial do livro de Rodrigo Tertulino, PhD — professor do IFRN, doutor em computação pela UFRGS e em engenharia informática pela Universidade de Coimbra.

[![Licença](https://img.shields.io/badge/licença-Uso%20Didático-blue)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-28.2.0-2496ED?logo=docker)](https://docs.docker.com)
[![Nmap](https://img.shields.io/badge/Nmap-7.99-0040FF)](https://nmap.org)
[![Snort](https://img.shields.io/badge/Snort-3.x-DC143C)](https://www.snort.org)

---

## Sobre o Livro

Este livro adota uma abordagem 100% prática no ensino de segurança de redes. Cada capítulo é estruturado em torno de laboratórios nos quais o leitor alterna entre os papéis de quem ataca e de quem defende uma rede. A progressão segue a metodologia **análise → defesa**, a mesma usada por profissionais de segurança em auditorias e testes de penetração reais.

**Fase de Análise (Capítulos 1–4):**
- Montagem do ambiente Docker com Kali Linux e Ubuntu
- Reconhecimento de rede com Nmap (descoberta de hosts, portas, versões, OS fingerprinting)
- Captura e análise de tráfego com Wireshark/tshark (handshake TCP, dados Redis em texto claro, credenciais HTTP)

**Fase de Defesa (Capítulos 5–8):**
- Firewall com iptables e nftables (políticas DROP, stateful inspection, NAT)
- Controle de acesso na camada de aplicação com Apache (ACLs por IP, autenticação HTTP básica, restrição de métodos)
- Detecção de intrusão com Snort 3 (regras para varredura Nmap, acesso ao Redis, força bruta SSH)
- Hardening de servidores Linux (SSH com chave pública, fail2ban, auth.log, mínimo privilégio)

**Apêndices:**
- Apêndice A — Próximos passos na carreira (certificações, cursos gratuitos, roteiro de estudos)
- Apêndice B — Referência rápida de comandos (Docker, Nmap, tcpdump, tshark, iptables, nftables, Apache, Redis, Snort, SSH)
- Apêndice C — Referências e recursos (LGPD, ISO 27001, NIST CSF, OWASP, documentações oficiais)

### Ferramentas utilizadas

| Ferramenta | Versão | Capítulo |
|---|---|---|
| Docker Engine | 28.2.0 | 1–8 |
| Nmap | 7.99 | 3, 8 |
| Wireshark / tshark | 4.6.6 | 4 |
| iptables / nftables | kernel 5.x+ | 5 |
| Apache HTTP Server | 2.4.52 | 6 |
| Snort 3 | 3.x | 7 |
| OpenSSH | 8.9p1 | 8 |
| fail2ban | 0.11.2 | 8 |
| Redis | 6.x | 2, 5, 7, 8 |
| Kali Linux (container) | kali-rolling | 1–8 |
| Ubuntu (container) | 22.04 LTS | 1–8 |

Ambiente validado em **macOS Sequoia 15 (Apple M4)** e **Docker Desktop 4.78**.

---

## Topologia do Laboratório

```
┌─────────────────────┐         ┌─────────────────────┐
│     ATACANTE         │         │       ALVO           │
│   Kali Linux         │         │   Ubuntu 22.04       │
│   192.168.10.10      │         │   192.168.10.20      │
│                      │         │                      │
│  • nmap              │  labnet │  • Redis   :6379     │
│  • tcpdump           │◄───────►│  • SSH     :22       │
│  • tshark            │         │  • Apache  :80       │
│  • snort             │         │  • iptables/nftables │
└─────────────────────┘         └─────────────────────┘
         │                              │
         └────────── 192.168.10.0/24 ───┘
              Docker Bridge Network
```

O contêiner **atacante** roda Kali Linux com ferramentas de análise ofensiva. O container **alvo** roda Ubuntu 22.04 com três serviços propositalmente vulneráveis para os laboratórios: Redis sem autenticação, SSH com login por senha e Apache com formulário HTTP sem TLS.

---

## Estrutura do Repositório

```
.
├── README.md
├── assets/
│   └── docker-compose.yml          # Ambiente de laboratório (2 containers)
├── configs/
│   ├── cap04-firewall/
│   │   ├── iptables-rules.v4       # Regras iptables (Laboratórios 1–3)
│   │   └── nftables.conf           # Regras nftables (Laboratório 4)
│   ├── cap05-acl/
│   │   └── 000-default.conf        # Virtual host Apache com ACLs
│   ├── cap06-snort/
│   │   └── local.rules             # Regras Snort 3: varredura, Redis, SSH
│   └── cap07-hardening/
│       ├── sshd_config             # Configuração endurecida do SSH
│       ├── ssh-banner.txt          # Banner de aviso legal
│       ├── fail2ban-jail.local     # Configuração do fail2ban
│       ├── auditd-hardening.rules  # Regras de auditoria (auditd)
│       └── sudoers-operador        # Configuração sudo com mínimo privilégio
```

### Download rápido dos arquivos de configuração

Todos os arquivos de configuração estão disponíveis para download direto. Substitua `<arquivo>` pelo caminho desejado:

```bash
# Exemplo: baixar docker-compose.yml
curl -LO https://github.com/rodrigoronner/seguranca-redes-pratica/raw/main/assets/docker-compose.yml

# Exemplo: baixar regras iptables
curl -LO https://github.com/rodrigoronner/seguranca-redes-pratica/raw/main/configs/cap04-firewall/iptables-rules.v4

# Exemplo: baixar sshd_config endurecido
curl -LO https://github.com/rodrigoronner/seguranca-redes-pratica/raw/main/configs/cap07-hardening/sshd_config
```

---

## Descrição dos Arquivos de Configuração

### `assets/docker-compose.yml`

Define a topologia completa do laboratório: rede virtual `labnet` (192.168.10.0/24), container atacante (Kali Linux, 192.168.10.10) e container alvo (Ubuntu 22.04, 192.168.10.20) com Redis, SSH e Apache pré-configurados.

> **Nota para Apple Silicon (M1/M2/M3/M4):** As linhas `platform: linux/arm64` já estão incluídas. Para processadores Intel/AMD (x86_64), remova essas linhas.

### `configs/cap04-firewall/iptables-rules.v4`

Conjunto completo de regras iptables com política DROP, incluindo: loopback, established/related, ICMP, bloqueio do Redis externo, SSH restrito ao atacante (192.168.10.10) e HTTP público. Compatível com `iptables-restore`.

### `configs/cap04-firewall/nftables.conf`

Mesmas regras do iptables, reescritas na sintaxe moderna do nftables, com conntrack (`ct state`). Compatível com `nft -f`.

### `configs/cap05-acl/000-default.conf`

Virtual host do Apache com três diretórios protegidos:
- `/` — público para todos
- `/admin` — restrito ao IP 192.168.10.10 (`Require ip`)
- `/interno` — bloqueado para todos (`Require all denied`)

O Laboratório 2 do capítulo estende este arquivo adicionando autenticação HTTP básica (`AuthType Basic` + `RequireAll`) e restrição de métodos HTTP com `LimitExcept`.

### `configs/cap06-snort/local.rules`

Regras personalizadas para Snort 3:
- `sid:1000001` — Ping ICMP na labnet
- `sid:1000002` — Varredura de porta SYN (flag S)
- `sid:1000003` — Acesso ao Redis (comando PING sem autenticação)
- `sid:1000004` — Escrita Redis (comando SET sem autenticação)

### `configs/cap07-hardening/`

| Arquivo | Descrição |
|---|---|
| `sshd_config` | SSH endurecido: `PasswordAuthentication no`, `PubkeyAuthentication yes`, `MaxAuthTries 3`, `PermitRootLogin prohibit-password`, banner legal |
| `ssh-banner.txt` | Aviso legal exibido antes do login SSH |
| `fail2ban-jail.local` | Jail SSH com `bantime=3600s`, `maxretry=3`, `findtime=600s` |
| `auditd-hardening.rules` | Monitora `/etc/passwd`, `/etc/shadow`, `sshd_config`, `sudoers` e comandos privilegiados |
| `sudoers-operador` | Usuário `operador` com permissão limitada para reiniciar o Apache e o SSH (sem senha) |

---

## Uso Rápido

### Pré-requisitos

- Docker Engine 28.x+ e Docker Compose v2.x
- Nmap 7.99+ (instalado no host para alguns labs)
- tshark 4.6+ (para análise de capturas)

### Iniciar o laboratório

```bash
# 1. Clone o repositório
git clone https://github.com/rodrigoronner/seguranca-redes-pratica.git
cd seguranca-redes-pratica

# 2. Suba os containers
docker compose up -d

# 3. Instale ferramentas no contêiner atacante
docker exec atacante apt-get update -qq
docker exec atacante apt-get install -y nmap tcpdump net-tools iputils-ping curl wget dnsutils redis-tools -qq

# 4. Verifique o ambiente
docker compose ps
docker exec atacante ping -c 4 192.168.10.20
docker exec atacante redis-cli -h 192.168.10.20 ping
docker exec atacante curl -s -o /dev/null -w '%{http_code}' http://192.168.10.20/
```

### Gerenciamento dos contêineres

```bash
docker compose stop       # Pausa os containers preservando dados
docker compose start      # Retoma do ponto onde parou
docker compose down       # Destrói containers e rede (dados são perdidos)
docker compose logs alvo  # Visualiza logs do container alvo
```

### Copiar arquivos para dentro/fora dos containers

```bash
docker cp atacante:/tmp/captura.pcap ~/seguranca-labs/   # Container -> Host
docker cp ./arquivo atacante:/tmp/                         # Host -> Container
```

---

## Conteúdo por Capítulo

| Capítulo | Tema | Ferramentas | Arquivos de configuração |
|---|---|---|---|
| 0 | Contexto, LGPD, ISO 27001, NIST CSF, OWASP | — | — |
| 1 | Ambiente Docker, Kali, Ubuntu | Docker | `assets/docker-compose.yml` |
| 2 | Varredura de rede | Nmap | — |
| 3 | Análise de tráfego | Wireshark, tshark, tcpdump | — |
| 4 | Firewall no Linux | iptables, nftables | `configs/cap04-firewall/` |
| 5 | Controle de acesso na aplicação | Apache | `configs/cap05-acl/` |
| 6 | Detecção de intrusão | Snort 3 | `configs/cap06-snort/` |
| 7 | Hardening de servidores | SSH, fail2ban, auditd, sudo | `configs/cap07-hardening/` |
| Apêndice A | Carreiras e certificações | — | — |
| Apêndice B | Referência rápida de comandos | — | — |
| Apêndice C | Referências e recursos | — | — |

---

## Aviso Legal

Este material é de uso exclusivamente didático. Todos os laboratórios utilizam ambientes isolados e controlados via Docker. Nenhum conteúdo deste livro ou deste repositório deve ser utilizado para atividades ilegais ou não autorizadas. O uso de ferramentas de segurança ofensiva sem autorização expressa do proprietário do sistema configura crime conforme o Art. 154-A do Código Penal Brasileiro (Lei nº 12.737/2012).

---

## Autor

**Rodrigo Tertulino**

- Professor IFRN — campus Mossoró
- Doutor em Computação (UFRGS) e Doutor em Engenharia Informática (Universidade de Coimbra)
- Coordenador do Laboratório de Pesquisa em Engenharia de Software e Automação (LaPEA)
- Instrutor Cisco Networking Academy e Huawei ICT Academy
- [Lattes](http://lattes.cnpq.br/5863705420808941) | [LinkedIn](https://www.linkedin.com/in/rodrigo-tertulino-phd-06557962/)

---

## Licença

© Rodrigo Tertulino. Este repositório contém material complementar ao livro *Segurança de Redes na Prática*. Uso didático permitido. Consulte o livro para termos completos.
