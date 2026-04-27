# TryHackMe — Brooklyn Nine Nine

![THM](https://img.shields.io/badge/Platform-TryHackMe-red)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-blue)
![OS](https://img.shields.io/badge/OS-Linux-orange)
![Category](https://img.shields.io/badge/Category-FTP%20%7C%20Steganography%20%7C%20Privilege%20Escalation-red)

---

# Informações da Máquina

| Nome | Dificuldade | Plataforma | OS |
| ---- | ---------- | ---------- | -- |
| Brooklyn Nine Nine | Easy | TryHackMe | Linux |

---

# Superfície de ataque

1. Enumeração inicial com Nmap  
2. Identificação de FTP anônimo  
3. Coleta de arquivo via FTP  
4. Enumeração web  
5. Identificação de pista sobre esteganografia  
6. Extração de credenciais escondidas em imagem  
7. Acesso via SSH como holt  
8. Enumeração de sudo  
9. Escalação de privilégio via nano  
10. Obtenção da flag root  

---

# Reconhecimento

A enumeração inicial foi realizada com Nmap para identificar portas abertas e serviços disponíveis na máquina alvo.

```
nmap -sC -sV -A -T4 10.64.137.31
```

![Nmap](/screenshots/nmap.png)

O scan revelou os seguintes serviços expostos:

- **21/tcp — FTP**
- **22/tcp — SSH**
- **80/tcp — HTTP**

O ponto mais interessante nessa etapa foi o FTP, pois o próprio Nmap identificou que o login anônimo estava habilitado:

```
ftp-anon: Anonymous FTP login allowed
```

Além disso, foi encontrado um arquivo chamado:

```
note_to_jake.txt
```

Esse tipo de arquivo costuma conter pistas importantes em máquinas CTF, então o próximo passo foi acessá-lo.

---

# Enumeração FTP

Como o FTP permitia autenticação anônima, foi possível baixar o arquivo diretamente usando `wget`.

```
wget ftp://anonymous:anonymous@10.64.137.31/note_to_jake.txt
```

![FTP File](/screenshots/ftp-file.png)

Depois de baixar o arquivo, visualizei seu conteúdo:

```
cat note_to_jake.txt
```

![Cat FTP File](/screenshots/cat-ftp-file.png)

O conteúdo era:

```
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

Essa mensagem revelou três informações importantes:

1. Existem possíveis usuários chamados **amy**, **jake** e **holt**
2. A senha de **jake** é fraca
3. O usuário **holt** provavelmente possui algum nível de importância na máquina

A princípio, isso poderia indicar um caminho por brute force contra o SSH do usuário `jake`. Porém, antes de partir para força bruta, continuei a enumeração web para procurar outras pistas.

---

# Enumeração Web

A porta 80 estava aberta, então acessei a aplicação web no navegador.

Durante a análise do código-fonte da página, encontrei um comentário interessante:

```
<!-- Have you ever heard of steganography? -->
```

![Source Code](/screenshots/source-code.png)

Esse comentário indicava que provavelmente havia alguma informação escondida dentro de uma imagem da página.

No CSS da aplicação, foi possível identificar que a imagem utilizada como background era:

```
brooklyn99.jpg
```

A partir disso, o próximo passo foi baixar a imagem e analisar se existia algum conteúdo oculto nela.

---

# Esteganografia

Como a própria aplicação deixava uma pista sobre steganography, utilizei o `stegseek` para tentar extrair dados escondidos da imagem usando a wordlist `rockyou.txt`.

```
stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

![Stegseek](/screenshots/stegseek.png)

O `stegseek` encontrou a passphrase:

```
admin
```

E extraiu um arquivo chamado:

```
brooklyn99.jpg.out
```

Ao visualizar o conteúdo do arquivo extraído:

```
cat brooklyn99.jpg.out
```

![Cat Stegseek](/screenshots/cat-stegseek.png)

Foi encontrada a senha do usuário `holt`:

```
Holt Password:
fluffydog12@ninenine

Enjoy!!
```

Nesse ponto, a exploração mudou de direção. Em vez de tentar força bruta no usuário `jake`, foi possível usar uma credencial válida encontrada via esteganografia.

---

# Acesso Inicial

Com a senha encontrada, tentei autenticar via SSH como o usuário `holt`.

```
ssh holt@10.64.137.31
```

![SSH](/screenshots/ssh.png)

A autenticação foi bem-sucedida, resultando em uma shell no sistema como:

```
holt
```

Esse acesso inicial foi obtido explorando uma cadeia simples:

1. Enumeração web
2. Comentário indicando esteganografia
3. Extração de credencial da imagem
4. Login via SSH

---

# Flag de Usuário

Após acessar a máquina como `holt`, foi possível ler a flag de usuário.

```
cat user.txt
```

![User Flag](/screenshots/user-flag.png)

```
ee11cbb19052e40b07aac0ca060c23ee
```

---

# Enumeração Local

Com acesso ao sistema, o próximo passo foi verificar permissões sudo do usuário atual.

```
sudo -l
```

![Sudo Nano](/screenshots/sudo-nano.png)

O resultado mostrou que o usuário `holt` podia executar o `nano` como qualquer usuário, sem senha:

```
User holt may run the following commands on brooklyn_nine_nine:
    (ALL) NOPASSWD: /bin/nano
```

Essa permissão é crítica, pois editores de texto como `nano`, `vim` e outros podem permitir execução de comandos dentro do próprio programa.

---

# Escalação de Privilégio

Como o usuário podia executar o `nano` como root, foi possível abusar dessa permissão para executar comandos privilegiados.

Primeiro, executei o `nano` com sudo:

```
sudo /bin/nano
```

Dentro do `nano`, é possível usar a função de execução de comandos.

O payload utilizado foi:

```
reset; sh 1>&0 2>&0
```

![Root Shell](/screenshots/root-shell.png)

Após executar o comando, foi obtida uma shell como root.

Para confirmar:

```
id
```

Resultado:

```text
uid=0(root) gid=0(root) groups=0(root)
```

A lógica por trás dessa exploração é simples: embora o sudo permita apenas executar `/bin/nano`, o próprio `nano` possui funcionalidades internas que podem chamar comandos do sistema. Como o programa estava sendo executado com privilégios de root, qualquer comando iniciado a partir dele também herdava esses privilégios.

---

# Root Shell

Com a shell elevada, o usuário atual passou a ser:

```
root
```

Isso confirmou o comprometimento total da máquina.

---

# Flag Root

Com privilégios de root, foi possível acessar o arquivo `root.txt`.

```
cat root.txt
```

![Root Flag](/screenshots/root-flag.png)

```
63a9f0ea7bb98050796b649e85481845
```

---

# Vulnerabilidades Identificadas

### FTP anônimo habilitado

O serviço FTP permitia login anônimo, possibilitando acesso ao arquivo `note_to_jake.txt`.

Esse arquivo não dava acesso direto à máquina, mas fornecia informações úteis sobre possíveis usuários e senhas fracas.

---

### Informação sensível escondida em imagem

A aplicação web continha uma pista no código-fonte sobre esteganografia.

A imagem `brooklyn99.jpg` possuía um arquivo oculto contendo a senha do usuário `holt`.

---

### Credenciais fracas/expostas

A senha do usuário `holt` foi encontrada dentro da imagem:

```
fluffydog12@ninenine
```

Com essa credencial, foi possível acessar a máquina via SSH.

---

### Configuração sudo insegura

O usuário `holt` podia executar o `nano` como root sem senha:

```
(ALL) NOPASSWD: /bin/nano
```

Essa configuração permitiu escalar privilégios para root através da execução de comandos dentro do próprio editor.

---

# Ferramentas Utilizadas

- Nmap
- FTP
- wget
- Stegseek
- rockyou.txt
- SSH
- sudo
- nano

---

# Principais Aprendizados

- Sempre verificar se FTP anônimo está habilitado
- Arquivos encontrados em serviços expostos podem conter pistas importantes
- Comentários em código-fonte podem revelar o caminho da exploração
- Esteganografia pode ser usada para esconder credenciais em imagens
- Permissões sudo aparentemente simples podem permitir escalação total
- Editores de texto executados com sudo devem ser tratados como risco crítico

---

# Autor

https://github.com/ninjaa-exe
