# Segurança de Redes na Prática

**Análise de Tráfego, Varredura, Firewall e Políticas de Acesso**

Repositório oficial do livro de Rodrigo Tertulino, PhD — Professor do IFRN, Doutor em Computação pela UFRGS e em Engenharia Informática pela Universidade de Coimbra.

[![Licença](https://img.shields.io/badge/licença-Uso%20Didático-blue)](LICENSE)
[![Docker](https://img.shields.io/badge/Docker-28.2.0-2496ED?logo=docker)](https://docs.docker.com)
[![Nmap](https://img.shields.io/badge/Nmap-7.99-0040FF?logo=nmap)](https://nmap.org)
[![Snort](https://img.shields.io/badge/Snort-3.x-DC143C)](https://www.snort.org)

---

## Sobre o Livro

Este livro adota uma abordagem 100% prática para o ensino de segurança de redes. Cada capítulo é estruturado em torno de laboratórios nos quais o leitor alterna entre os papéis de quem ataca e de quem defende uma rede. A progressão segue a metodologia **análise → defesa**, a mesma usada por profissionais de segurança em auditorias e testes de penetração reais.

**Fase de Análise (Capítulos 1–4):**
- Montagem do ambiente Docker com Kali Linux e Ubuntu
- Reconhecimento de rede com Nmap (descoberta de hosts, portas, versões, OS fingerprinting)
- Captura e análise de tráfego com Wireshark/tshark (handshake TCP, dados Redis em texto claro, credenciais HTTP)

**Fase de Defesa (Capítulos 5–8):**
- Firewall com iptables e nftables (políticas DROP, stateful inspection, NAT)
- Controle de acesso na camada de aplicação com Apache (ACLs por IP, autenticação HTTP básica, restrição de métodos)
- Detecção de intrusão com Snort 3 (regras para varredura Nmap, acesso ao Redis, força bruta SSH)
- Hardening de servidores Linux (SSH com chave pública, fail2ban, auditd, mínimo privilégio)

**Apêndices:**
- Apêndice A — Próximos passos na carreira (certificações, cursos gratuitos, roteiro de estudos)
- Apêndice B — Referência rápida de comandos (Docker, Nmap, tcpdump, tshark, iptables, nftables, Apache, Redis, Snort, SSH, auditd)
- Apêndice C — Referências e recursos (LGPD, ISO 27001, NIST CSF, OWASP, documentações oficiais)

### Ferramentas utilizadas

| Ferramenta | Versão | Capítulo |
|-----------|--------|----------|
| Docker Engine | 28.2.0 | 1–8 |
| Nmap | 7.99 | 2, 7 |
| Wireshark / tshark | 4.6.6 | 3 |
| iptables / nftables | — | 4 |
| Apache HTTP Server | 2.4.52 | 5 |
| Snort 3 | 3.x | 6 |
| OpenSSH / fail2ban / auditd | — | 7 |
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
│  • snort             │         │  • iptables          │
└─────────────────────┘         └─────────────────────┘
         │                              │
         └────────── 192.168.10.0/24 ───┘
              Docker Bridge Network
```

O container **atacante** roda Kali Linux com ferramentas de análise ofensiva. O container **alvo** roda Ubuntu 22.04 com três serviços propositalmente vulneráveis para os laboratórios: Redis sem autenticação, SSH com login por senha e Apache com formulário HTTP sem TLS.

---

## Estrutura do Repositório

```
.
├── docker-compose.yml              # Ambiente de laboratório (2 containers)
├── README.md                       # Este arquivo
├── cap04-firewall/
│   ├── iptables-rules.v4           # Regras iptables (Laboratórios 1–3)
│   └── nftables.conf               # Regras nftables (Laboratório 4)
├── cap05-acl/
│   └── 000-default.conf            # Virtual host Apache com ACLs (Laboratórios 1–3)
├── cap06-snort/
│   └── local.rules                 # Regras Snort 3: varredura, Redis, SSH
└── cap07-hardening/
    ├── sshd_config                 # Configuração endurecida do SSH
    ├── ssh-banner.txt              # Banner de aviso legal
    ├── fail2ban-jail.local         # Configuração do fail2ban
    ├── auditd-hardening.rules      # Regras de auditoria (auditd)
    └── sudoers-operador            # Configuração sudo com mínimo privilégio
```

### Descrição dos arquivos

#### `docker-compose.yml`
Define a topologia completa do laboratório: rede virtual `labnet` (192.168.10.0/24), container atacante (192.168.10.10) com Kali Linux, e container alvo (192.168.10.20) com Ubuntu 22.04, Redis, SSH e Apache pré-configurados.

> **Nota para Apple Silicon (M1/M2/M3/M4):** As linhas `platform: linux/arm64` já estão incluídas. Para processadores Intel/AMD (x86_64), remova essas linhas.

#### `cap04-firewall/iptables-rules.v4`
Conjunto completo de regras iptables com política DROP, incluindo: loopback, established/related, ICMP, bloqueio do Redis externo, SSH restrito por IP e HTTP público. Compatível com `iptables-restore`.

#### `cap04-firewall/nftables.conf`
Mesmas regras do iptables reescritas na sintaxe moderna do nftables com conntrack. Compatível com `nft -f`.

#### `cap05-acl/000-default.conf`
Virtual host do Apache com três diretórios protegidos:
- `/` → público
- `/admin` → IP 192.168.10.10 + autenticação básica (RequireAll)
- `/interno` → bloqueado (Require all denied)
- `/api` → GET livre, escrita restrita (LimitExcept)

#### `cap06-snort/local.rules`
Regras personalizadas para Snort 3:
- `sid:1000001` — Ping ICMP para o alvo
- `sid:1000002` — Varredura de porta SYN (>15 pacotes/s)
- `sid:1000003` — Acesso Redis (comando PING)
- `sid:1000004` — Escrita Redis (comando SET)

#### `cap07-hardening/`
- **sshd_config** — SSH endurecido: PermitRootLogin no, PasswordAuthentication no, PubkeyAuthentication yes, MaxAuthTries 3, banner legal
- **ssh-banner.txt** — Banner de acesso restrito com aviso legal
- **fail2ban-jail.local** — Jail SSH com bantime de 3600s e maxretry de 3
- **auditd-hardening.rules** — Monitora /etc/passwd, /etc/shadow, sshd_config, sudoers e comandos privilegiados
- **sudoers-operador** — Usuário operador com permissão limitada a restart do Apache e SSH

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
docker exec atacante curl -s -o /dev/null -w "%{http_code}" http://192.168.10.20/
```

### Gerenciamento

```bash
docker compose stop       # Pausa os containers preservando dados
docker compose start      # Retoma do ponto onde parou
docker compose down       # Destrói containers e rede (dados são perdidos)
docker compose logs alvo  # Visualiza logs do container alvo
```

### Copiar arquivos para dentro/fora dos containers

```bash
docker cp atacante:/tmp/captura.pcap ~/seguranca-labs/   # Container → Host
docker cp ./arquivo atacante:/tmp/                         # Host → Container
```

---

## Conteúdo por Capítulo

| Capítulo | Tema | Ferramentas | Arquivos |
|----------|------|-------------|----------|
| 1 | Contexto, LGPD, ISO 27001, NIST CSF, OWASP | — | — |
| 2 | Ambiente Docker, Kali, Ubuntu | Docker, Kali, Ubuntu | `docker-compose.yml` |
| 3 | Varredura de rede | Nmap | — |
| 4 | Análise de tráfego | Wireshark, tshark, tcpdump | — |
| 5 | Firewall no Linux | iptables, nftables | `iptables-rules.v4`, `nftables.conf` |
| 6 | Controle de acesso na aplicação | Apache | `000-default.conf` |
| 7 | Detecção de intrusão | Snort 3 | `local.rules` |
| 8 | Hardening de servidores | SSH, fail2ban, auditd, sudo | `sshd_config`, `fail2ban-jail.local`, `auditd-hardening.rules`, `sudoers-operador` |

---

## Aviso Legal

Este material é de uso exclusivamente didático. Todos os laboratórios utilizam ambientes isolados e controlados via Docker. Nenhum conteúdo deste livro ou deste repositório deve ser utilizado para atividades ilegais ou não autorizadas. O uso de ferramentas de segurança ofensiva sem autorização expressa do proprietário do sistema configura crime conforme o Art. 154-A do Código Penal Brasileiro (Lei nº 12.737/2012).

---

## Autor

**Rodrigo Tertulino, PhD**

- Professor EBTT do IFRN — campus Mossoró
- Doutor em Computação (UFRGS) e Doutor em Engenharia Informática (Universidade de Coimbra)
- Coordenador do Laboratório de Pesquisa em Engenharia de Software e Automação (LaPEA)
- Instrutor Cisco Networking Academy e Huawei ICT Academy
- [Lattes](http://lattes.cnpq.br/5863705420808941) | [LinkedIn](https://www.linkedin.com/in/rodrigo-tertulino-phd-06557962/)

---

## Licença

© Rodrigo Tertulino. Este repositório contém material complementar ao livro *Segurança de Redes na Prática*. Uso didático permitido. Consulte o livro para termos completos.
