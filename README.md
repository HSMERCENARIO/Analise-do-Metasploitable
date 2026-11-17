# Homelab de Cibersegurança - Análise do Metasploitable 2

## 1. Objetivo do Projeto

Este laboratório foi criado com o objetivo de desenvolver na prática minhas habilidades em cibersegurança, especialmente nas etapas de reconhecimento e exploração de vulnerabilidades.

Além de praticar, o foco também foi documentar todo o processo de aprendizado, desde a montagem do ambiente até a análise pós-exploração, de forma organizada e clara.

## 2. Ambiente Utilizado

- **Sistema operacional do host:** Windows 10
- **Ferramenta de virtualização:** VirtualBox 7.2.20
- **Máquinas virtuais usadas:**
  - Kali Linux 2026.3 (máquina atacante)
  - Metasploitable 2 (máquina alvo, vulnerável por design)
- **Configuração de rede:** Host-Only Adapter, isolando o laboratório da rede externa

## 3. Configuração do Laboratório

Utilizei o VirtualBox para instalar o Kali Linux e o Metasploitable 2 em um ambiente isolado e seguro.

Configurei a rede de ambas as máquinas virtuais como **Host-Only Adapter**, garantindo que elas se comunicassem apenas entre si e com o host, sem acesso à internet externa. Essa configuração cria um laboratório controlado e seguro para a prática de testes de segurança.

## 4. Reconhecimento (Scanning)

Utilizei o Nmap para mapear os serviços em execução na máquina alvo, combinando detecção de versão (`-sV`) com os scripts padrão de verificação (`-sC`).

### Comando utilizado
```bash
nmap -sV -sC -n 192.168.56.102
```

A flag `-n` foi usada para desativar a resolução de DNS reverso, evitando um bug conhecido nessa versão do Nmap.

### Portas encontradas

| Porta | Serviço | Versão |
|-------|---------|--------|
| 21 | ftp | vsftpd 2.3.4 |
| 22 | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 23 | telnet | Linux telnetd |
| 25 | smtp | Postfix smtpd |
| 80 | http | Apache httpd 2.2.8 |
| 111 | rpcbind | RPC #100000 |
| 139 | netbios-ssn | Samba smbd 3.X - 4.X |
| 445 | netbios-ssn | Samba smbd 3.X - 4.X |
| 512 | exec | netkit-rsh rexecd |
| 513 | login | OpenBSD/Solaris rlogin |
| 514 | shell | Netkit rshd |
| 1099 | java-rmi | GNU Classpath grmiregistry |
| 1524 | bindshell | Metasploitable root shell |
| 2049 | nfs | NFS #100003 |
| 2121 | ftp | ProFTPD 1.3.1 |
| 5900 | vnc | VNC (protocol 3.3) |
| 6000 | X11 | acesso negado |
| 6667 | irc | UnrealIRCd |

### O que cada porta representa

- **21 (FTP):** transferência de arquivos. Essa versão (vsftpd 2.3.4) ficou famosa por conter um backdoor.
- **22 (SSH):** acesso remoto seguro ao terminal.
- **23 (Telnet):** acesso remoto ao terminal **sem criptografia**, tudo trafega em texto puro.
- **25 (SMTP):** serviço de envio de e-mails.
- **80 (HTTP):** servidor web.
- **111 (rpcbind):** serviço que gerencia chamadas RPC, usado por outros serviços como o NFS.
- **139/445 (Samba):** compartilhamento de arquivos e impressoras no padrão Windows (SMB).
- **512/513/514 (rexec/rlogin/rsh):** comandos remotos antigos, sem criptografia e hoje considerados inseguros.
- **1099 (Java RMI):** permite chamadas remotas de métodos Java, pode ser abusado para executar código.
- **1524 (bindshell):** porta com um shell de root aberto sem autenticação. Detalhada na Seção 5.
- **2049 (NFS):** compartilhamento de arquivos entre sistemas Unix/Linux.
- **2121 (ProFTPD):** outro servidor FTP, versão diferente do da porta 21.
- **5900 (VNC):** acesso remoto gráfico (área de trabalho).
- **6000 (X11):** sistema gráfico do Linux, aqui com acesso negado.
- **6667 (IRC):** chat em tempo real. Esse serviço (UnrealIRCd) também tem uma vulnerabilidade famosa de backdoor.

## 5. Exploração da Porta 1524

### O que é a porta 1524

A porta 1524/tcp é originalmente usada pelo serviço `ingreslock`, mas no Metasploitable 2 ela foi deixada de propósito com um **backdoor**: um shell de root escutando diretamente nessa porta, sem nenhuma autenticação. Qualquer conexão recebida nela já entrega acesso total ao sistema.

### Como consegui acesso

Utilizei o Netcat para conectar diretamente na porta:

```bash
nc 192.168.56.102 1524
```

Assim que a conexão foi estabelecida, o terminal já retornou um prompt de root, sem pedir usuário ou senha:

```
root@metasploitable:/#
```

### Comprovação de que entrei como root

Para confirmar o nível de acesso, executei:

```bash
whoami
```

O retorno foi `root`, confirmando acesso total ao sistema.

## 6. Pós-Exploração

Após confirmar o acesso root através do `whoami`, explorei o sistema para identificar informações sensíveis que só seriam acessíveis com esse nível de privilégio.

### Comandos utilizados

```bash
whoami
cat /etc/passwd
cat /etc/shadow
```

### Principais descobertas

- Consegui visualizar o conteúdo completo dos arquivos `/etc/passwd` e `/etc/shadow` **sem nenhuma restrição**, o que comprova o acesso irrestrito obtido através do backdoor da porta 1524.
- O `/etc/passwd` lista os usuários cadastrados no sistema, com informações como nome de usuário, UID, GID, diretório home e shell padrão, mas sem as senhas em si.
- O `/etc/shadow` contém os **hashes** das senhas de cada usuário, um arquivo que normalmente só o root tem permissão de ler. O fato de ter sido acessado livremente demonstra o nível crítico da falha explorada.

### Tentativa de quebra de hashes (John the Ripper)

Tentei utilizar o John the Ripper para quebrar os hashes extraídos do `/etc/shadow`, porém não foi possível concluir o processo devido às limitações de hardware da máquina utilizada no laboratório. Essa etapa fica registrada como próximo passo do projeto (Seção 8).

## 7. O que aprendi

Aprendi a utilizar o Nmap de forma mais aprofundada, empregando a flag `-sV` para identificar os serviços e suas versões, além da flag `-n` para evitar um bug relacionado à resolução de DNS.

Também aprendi na prática como explorar a vulnerabilidade da porta 1524 e compreendi os riscos de deixar um serviço desse tipo exposto sem a devida proteção, já que ele permite acesso root completo sem qualquer autenticação.

## 8. Próximos passos

Como continuidade deste projeto, pretendo:

- Rodar o John the Ripper em um teste futuro, utilizando uma máquina com mais capacidade de processamento, para tentar quebrar os hashes extraídos do `/etc/shadow`.
- Explorar a porta 6667 (UnrealIRCd), que também apresenta uma vulnerabilidade conhecida identificada na etapa de reconhecimento.
- Corrigir a falha da porta 1524, aplicando as devidas medidas de mitigação, como forma de praticar também o lado defensivo (blue team) além do ofensivo.

## 9. Conclusão

Este projeto permitiu colocar em prática, de forma controlada e segura, conceitos fundamentais de cibersegurança que até então eu conhecia apenas de forma teórica. Desde a montagem do laboratório com VirtualBox, Kali Linux e Metasploitable 2, passando pelo reconhecimento de serviços com o Nmap, até a exploração de uma vulnerabilidade crítica na porta 1524, cada etapa reforçou a importância de entender não apenas **como** executar um ataque, mas **por que** ele funciona e **quais riscos** ele representa.

A facilidade com que obtive acesso root através de um serviço mal configurado, sem qualquer autenticação, deixou clara a importância de práticas básicas de segurança, como manter softwares atualizados, desativar serviços desnecessários e revisar periodicamente quais portas estão expostas na rede.

Além do aprendizado técnico, este projeto também me ensinou a documentar um processo de forma clara e organizada, uma habilidade tão importante quanto a execução técnica em si, especialmente na área de segurança da informação, onde registrar o que foi feito é parte essencial do trabalho.

Pretendo continuar evoluindo este laboratório, aprofundando tanto no lado ofensivo quanto no defensivo, como descrito na seção de próximos passos.
