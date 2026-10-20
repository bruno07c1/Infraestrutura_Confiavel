Infraestrutura de Rede Confiável: DHCP, DNS, NTP e Syslog

Laboratório de redes com foco em cibersegurança, montado no Cisco Packet Tracer. Duas redes separadas (clientes e servidores), serviços de infraestrutura centralizados e dispositivos de rede sincronizados por NTP, enviando logs a um servidor Syslog.

Sumário
Visão geral
Por que isso importa para a segurança
Ferramentas e requisitos
Topologia
Plano de endereçamento
Os serviços em detalhe
Configurações
Como reproduzir passo a passo
Testes e validação
Demonstração: por que a hora importa
Análise de segurança
Problemas encontrados e soluções
Limitações do Packet Tracer
Melhorias futuras
Conceitos aprendidos
Estrutura do repositório
Referências
Autor
1. Visão geral

Este projeto monta a base de infraestrutura que quase toda rede corporativa precisa antes de qualquer controle de segurança mais avançado:

Serviço	Função no projeto
DHCP	Entrega IP, máscara, gateway e DNS automaticamente aos PCs, mesmo com o servidor em outra rede (DHCP relay)
DNS	Traduz nomes (ntp.lab.local) em endereços IP
NTP	Mantém o relógio de todos os dispositivos sincronizado
Syslog	Centraliza os logs do roteador e dos switches em um só lugar
Objetivos
Separar clientes e servidores em redes distintas, interligadas por um roteador.
Configurar DHCP relay (ip helper-address) para atender clientes de outra sub-rede.
Sincronizar todos os equipamentos com um servidor NTP.
Enviar logs dos equipamentos a um servidor Syslog central.
Demonstrar na prática por que relógios sincronizados são essenciais para investigar incidentes.
Documentar o projeto de forma que outra pessoa consiga reproduzi-lo.
Resumo do que foi construído
1 roteador, 2 switches, 3 PCs e 4 servidores.
2 redes IPv4 /24: 192.168.10.0/24 (clientes) e 192.168.20.0/24 (servidores).
4 servidores dedicados, um para cada serviço.
2. Por que isso importa para a segurança

Estes quatro serviços parecem "só infraestrutura", mas são alvos e ferramentas de segurança importantes.

DHCP

Define como os dispositivos entram na rede. Um servidor DHCP falso pode entregar um gateway ou DNS malicioso aos clientes. Também permite saber qual IP estava com qual dispositivo em determinado momento, informação básica em uma investigação.

DNS

Quase todo acesso começa com uma consulta DNS. Se o DNS é manipulado (DNS spoofing, cache poisoning), o usuário vai parar em um destino falso sem perceber. Logs de DNS também ajudam a detectar comunicação com domínios maliciosos.

NTP

A hora é um dado de segurança. Sem relógios sincronizados:

não é possível reconstruir a linha do tempo de um incidente;
logs de dispositivos diferentes ficam impossíveis de correlacionar (um SIEM depende disso);
registros perdem valor como evidência;
mecanismos que dependem de horário, como certificados e autenticação, podem falhar.
Syslog

Logs locais podem ser apagados por quem invade o equipamento. Enviá-los para um servidor central preserva o histórico e permite analisar vários dispositivos em um só lugar. Esse é o primeiro passo para o que um SOC faz no dia a dia.

Ideia central do projeto: sem DHCP e DNS organizados a rede não funciona direito, e sem NTP e Syslog ela até funciona, mas não pode ser investigada quando algo dá errado.

3. Ferramentas e requisitos
Item	Detalhe
Simulador	Cisco Packet Tracer (versão: [PREENCHER, ex.: 8.2.2])
Roteador	Cisco 2911 (ou equivalente)
Switches	Cisco 2960
Servidores	Server-PT
Clientes	PC-PT
Cabos	Cobre direto (Copper Straight-Through)

Para abrir o projeto, baixe o Packet Tracer (gratuito com conta na Cisco Networking Academy) e abra o arquivo infraestrutura-confiavel.pkt.

4. Topologia

<img width="1152" height="713" alt="projeto" src="https://github.com/user-attachments/assets/af599f79-c020-4b02-9371-9a069d77ce51" />


Dispositivos
Dispositivo	Tipo	Função
R1	Roteador 2911	Interliga as duas redes e faz o DHCP relay
SW-CLI	Switch 2960	Conecta os PCs (rede de clientes)
SW-SRV	Switch 2960	Conecta os servidores (rede de servidores)
PC1, PC2, PC3	PC-PT	Clientes que recebem IP por DHCP
SRV-DHCP	Server-PT	Serviço DHCP
SRV-DNS	Server-PT	Serviço DNS
SRV-NTP	Server-PT	Serviço NTP
SRV-SYSLOG	Server-PT	Serviço Syslog
Cabeamento

Todos os cabos são cobre direto.

Origem	Porta	Destino	Porta
R1	GigabitEthernet0/0	SW-CLI	FastEthernet0/1
R1	GigabitEthernet0/1	SW-SRV	FastEthernet0/1
PC1	FastEthernet0	SW-CLI	FastEthernet0/2
PC2	FastEthernet0	SW-CLI	FastEthernet0/3
PC3	FastEthernet0	SW-CLI	FastEthernet0/4
SRV-DHCP	FastEthernet0	SW-SRV	FastEthernet0/2
SRV-DNS	FastEthernet0	SW-SRV	FastEthernet0/3
SRV-NTP	FastEthernet0	SW-SRV	FastEthernet0/4
SRV-SYSLOG	FastEthernet0	SW-SRV	FastEthernet0/5

Se o seu modelo de roteador tiver outros nomes de porta (por exemplo g0/0/0 ou fa0/0), use os nomes que aparecem em show ip interface brief.

Por que duas redes separadas?

Separar clientes e servidores é uma prática básica de segmentação. Facilita aplicar regras de acesso entre as redes (ACLs, nos próximos projetos), limita o alcance de um problema e reflete como redes reais são organizadas. Também permite praticar o DHCP relay, já que o servidor DHCP fica em uma rede diferente da dos clientes.

5. Plano de endereçamento
Redes
Rede	Endereço	Máscara	Gateway
Clientes	192.168.10.0/24	255.255.255.0	192.168.10.1
Servidores	192.168.20.0/24	255.255.255.0	192.168.20.1
Dispositivos
Dispositivo	IP	Máscara	Gateway	Atribuição
R1 G0/0 (clientes)	192.168.10.1	255.255.255.0	n/a	Estático
R1 G0/1 (servidores)	192.168.20.1	255.255.255.0	n/a	Estático
SW-CLI (VLAN 1)	192.168.10.2	255.255.255.0	192.168.10.1	Estático
SW-SRV (VLAN 1)	192.168.20.2	255.255.255.0	192.168.20.1	Estático
SRV-DHCP	192.168.20.10	255.255.255.0	192.168.20.1	Estático
SRV-DNS	192.168.20.11	255.255.255.0	192.168.20.1	Estático
SRV-NTP	192.168.20.12	255.255.255.0	192.168.20.1	Estático
SRV-SYSLOG	192.168.20.13	255.255.255.0	192.168.20.1	Estático
PC1, PC2, PC3	192.168.10.100 em diante	255.255.255.0	192.168.10.1	DHCP

Os servidores usam endereços estáticos, porque outros dispositivos dependem de saber onde eles estão. Os PCs recebem endereços dinâmicos.

6. Os serviços em detalhe
6.1 DHCP (SRV-DHCP)

Pool configurado:

Parâmetro	Valor
Nome do pool	CLIENTES
Gateway padrão	192.168.10.1
Servidor DNS	192.168.20.11
IP inicial	192.168.10.100
Máscara	255.255.255.0
Máximo de usuários	100

DHCP relay. Quando um PC liga, ele envia um broadcast pedindo um IP. Roteadores não repassam broadcasts entre redes. Como o servidor DHCP está na rede 192.168.20.0/24 e os PCs na 192.168.10.0/24, o R1 recebe o pedido e o encaminha ao servidor como unicast usando o comando:

interface g0/0
 ip helper-address 192.168.20.10

O comando fica na interface do lado dos clientes, apontando para o IP do servidor DHCP.

6.2 DNS (SRV-DNS)
Nome	Tipo	Endereço
ntp.lab.local	A	192.168.20.12
syslog.lab.local	A	192.168.20.13
dhcp.lab.local	A	192.168.20.10

Os clientes recebem o endereço deste servidor DNS pelo DHCP, então não precisam configurá-lo manualmente.

6.3 NTP (SRV-NTP)

O servidor NTP do Packet Tracer fornece a hora aos dispositivos. O roteador e os switches são clientes NTP:

ntp server 192.168.20.12

O fuso horário usado é BRT (UTC-3):

clock timezone BRT -3

O NTP usa a porta UDP 123. A hierarquia de precisão é organizada em níveis (stratum): o nível 0 são fontes de tempo precisas (relógios atômicos ou GPS), e cada nível abaixo sincroniza com o de cima.

6.4 Syslog (SRV-SYSLOG)

Os dispositivos enviam suas mensagens de log ao servidor:

logging host 192.168.20.13

Também foram ativados timestamps com data, hora e milissegundos:

service timestamps log datetime msec

O Syslog usa a porta UDP 514. Cada mensagem tem um nível de severidade:

Nível	Nome	Exemplo
0	Emergencies	Sistema inutilizável
1	Alerts	Ação imediata necessária
2	Critical	Condição crítica
3	Errors	Erros
4	Warnings	Avisos
5	Notifications	Eventos normais, mas relevantes (mudança de link, configuração alterada)
6	Informational	Informações
7	Debugging	Depuração

O nível padrão de envio (informational) cobre as mensagens de interface (%LINK-5-CHANGED, %LINEPROTO-5-UPDOWN) e de alteração de configuração (%SYS-5-CONFIG_I).

7. Configurações

As configurações completas exportadas de cada equipamento estão na pasta configs/:

Arquivo	Conteúdo
R1.txt	show running-config do roteador
SW-CLI.txt	show running-config do switch dos clientes
SW-SRV.txt	show running-config do switch dos servidores
servidores.md	Configuração dos servidores (sem CLI no Packet Tracer)
Resumo dos comandos principais
<details> <summary><b>Roteador R1</b></summary>
enable
configure terminal
hostname R1
clock timezone BRT -3
service timestamps log datetime msec
service timestamps debug datetime msec
interface g0/0
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 192.168.20.10
 no shutdown
exit
interface g0/1
 ip address 192.168.20.1 255.255.255.0
 no shutdown
exit
ntp server 192.168.20.12
logging host 192.168.20.13
end
copy running-config startup-config
</details> <details> <summary><b>Switch SW-SRV</b></summary>
enable
configure terminal
hostname SW-SRV
clock timezone BRT -3
service timestamps log datetime msec
interface vlan 1
 ip address 192.168.20.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.20.1
ntp server 192.168.20.12
logging host 192.168.20.13
end
copy running-config startup-config
</details> <details> <summary><b>Switch SW-CLI</b></summary>
enable
configure terminal
hostname SW-CLI
clock timezone BRT -3
service timestamps log datetime msec
interface vlan 1
 ip address 192.168.10.2 255.255.255.0
 no shutdown
exit
ip default-gateway 192.168.10.1
ntp server 192.168.20.12
logging host 192.168.20.13
end
copy running-config startup-config
</details>

O IP na VLAN 1 do switch serve apenas para gerência: ele permite que o switch converse com os servidores NTP e Syslog. Não é usado para encaminhar o tráfego dos PCs.

8. Como reproduzir passo a passo
Monte a topologia com os dispositivos e cabos das tabelas da seção 4. Aguarde as bolinhas dos cabos ficarem verdes.
Configure o roteador R1 (seção 7), incluindo no shutdown nas interfaces e o ip helper-address.
Configure os IPs estáticos dos servidores em Desktop → IP Configuration, conforme a seção 5.
Ative os serviços em Services de cada servidor:
SRV-DNS: serviço DNS On e registros A cadastrados.
SRV-DHCP: serviço DHCP On, pool CLIENTES editado e salvo (Save).
SRV-NTP: serviço NTP On.
SRV-SYSLOG: serviço SYSLOG On.
Configure os switches (IP de gerência, gateway, NTP e Syslog).
Configure os PCs para DHCP em Desktop → IP Configuration → DHCP.
Valide tudo seguindo a seção 9.
Salve as configurações (copy running-config startup-config) e o arquivo .pkt.
9. Testes e validação

Marque cada teste como concluído apenas depois de executá-lo. A coluna Evidência aponta para o print correspondente.

#	Teste	Comando ou ação	Resultado esperado	Status	Evidência
1	Interfaces do roteador	show ip interface brief no R1	Duas interfaces up/up com os IPs corretos	☐	n/a
2	DHCP via relay	PC → IP Configuration → DHCP	IP 192.168.10.100+, gateway 192.168.10.1, DNS 192.168.20.11	☐	dhcp-pc.png
3	Conectividade entre redes	ping 192.168.20.11 a partir de um PC	Resposta (reply)	☐	ping-dns.png
4	Resolução DNS	nslookup ntp.lab.local em um PC	Retorna 192.168.20.12	☐	ping-dns.png
5	Sincronização NTP	show ntp status e show ntp associations no R1	Relógio sincronizado com 192.168.20.12	☐	ntp-status.png
6	Envio de logs	shutdown e no shutdown em uma interface	Mensagens aparecem em SRV-SYSLOG → Services → SYSLOG	☐	syslog-server.png

Evidências

DHCP recebido pelo PC

<img width="1152" height="705" alt="pcDHCP" src="https://github.com/user-attachments/assets/700a747f-55a7-474d-a0e5-74ac052d40fe" />

Ping e DNS

<img width="1152" height="648" alt="pcPROMPT" src="https://github.com/user-attachments/assets/8d46879c-a3fd-4974-a461-5f881aa3680c" />

NTP sincronizado

<img width="1152" height="648" alt="ntpPrint" src="https://github.com/user-attachments/assets/bce0591b-1ed5-49f7-b6c7-044dba9eba7c" />

Logs no servidor Syslog

<img width="1152" height="705" alt="log" src="https://github.com/user-attachments/assets/f22fd3a6-b0dd-47cc-acf2-f1c2e4fddc37" />

10. Demonstração: por que a hora importa

Esta parte mostra o problema que o NTP resolve.

Cenário

Um incidente afeta dois equipamentos. Para entender o que aconteceu primeiro, é preciso ordenar os eventos dos dois logs.

Procedimento
Sem NTP: deixe o R1 e o SW-SRV com horários diferentes (por exemplo, usando clock set no modo #), com o NTP ainda desativado ou desassociado.
Gere um evento em cada um, na ordem R1 primeiro, SW-SRV depois.
Observe no Syslog que os horários registrados não refletem a ordem real.
Com NTP: ative a sincronização nos dois, repita o teste e compare.
O que observar
Situação	Resultado
Relógios diferentes	Os eventos aparecem fora de ordem ou com horas incoerentes, impossibilitando reconstruir a sequência
Relógios sincronizados	Os horários são coerentes e a linha do tempo pode ser reconstruída
Conclusão

Em uma investigação real, correlacionar logs de firewall, servidores, switches e aplicações exige que todos usem a mesma referência de tempo. Sem isso, a sequência de um ataque não pode ser reconstruída com confiança e o valor dos registros como evidência diminui.

Preencha aqui com suas observações e prints do experimento: [PREENCHER]

11. Análise de segurança
11.1 Riscos relacionados e controles
Serviço	Ameaça	Impacto	Controle (neste ou em projetos futuros)
DHCP	Servidor DHCP falso (rogue DHCP)	Clientes recebem gateway ou DNS malicioso	DHCP Snooping (projeto futuro)
DHCP	Esgotamento do pool (starvation)	Negação de serviço aos clientes	Port security, limite de MACs por porta (projeto futuro)
DNS	Resposta falsa (spoofing, cache poisoning)	Usuário vai para um destino falso	Servidor DNS confiável; DNSSEC em ambientes reais
NTP	Servidor de horário falso	Relógios alterados, logs inconsistentes	Autenticação NTP; restrição de acesso à porta UDP 123
Syslog	Mensagens falsas ou interceptação (UDP sem criptografia)	Logs enganosos ou expostos	Restringir a origem por ACL; transporte seguro em ambientes reais
Gerência	IP de gerência na VLAN 1	Superfície de ataque maior	VLAN de gerência dedicada
11.2 O que este projeto ainda não protege

Este projeto é uma base, e é importante ser claro sobre o que falta:

Não há ACLs controlando quem acessa os servidores.
O NTP não tem autenticação habilitada.
O Syslog usa UDP, sem criptografia nem garantia de entrega.
Os switches usam a VLAN 1 para gerência.
As portas dos switches não têm port security.
Não há SSH nem senhas nas linhas VTY (a configuração login sem senha impede o acesso remoto, mas não é uma configuração de segurança planejada).

Cada ponto acima vira tema de um projeto seguinte (ver seção 14).

12. Problemas encontrados e soluções

Registro dos problemas reais durante a montagem, que podem ajudar quem for reproduzir o laboratório.

Problema	Causa	Solução
DHCP request failed no PC	Faltava o ip helper-address na interface dos clientes, ou o pool do servidor não estava salvo	Adicionar o ip helper-address 192.168.20.10 na interface g0/0 e clicar em Save no pool
% Invalid input detected at '^' marker	Comando digitado no modo errado (ex.: enable já dentro de (config)#) ou erro de digitação (adress em vez de address)	Conferir o prompt, voltar com end e redigitar
% Invalid interface type and number	Comando de roteador aplicado em um switch, ou nome de porta diferente no modelo usado	Conferir os nomes reais com show ip interface brief
logging trap informational rejeitado	Comando não aceito na versão usada	Comando opcional; o nível padrão já é informational. Usar logging host ou logging 192.168.20.13
terminal length 0 rejeitado	Modo errado ou comando não suportado	Usar do terminal length 0 em (config)# ou avançar o --More-- com a barra de espaço
Bolinha laranja depois de shutdown e no shutdown	O switch recalcula o spanning tree na porta	Aguardar ou usar o fast forward do Packet Tracer
Print de tela "sumiu"	O atalho copiou a imagem para a área de transferência sem salvar	Colar no Paint e salvar, ou usar a Ferramenta de Captura
13. Limitações do Packet Tracer
O Server-PT não tem CLI, então a configuração dos servidores é documentada manualmente em configs/servidores.md.
O servidor NTP do Packet Tracer usa o relógio do próprio simulador, não uma fonte externa real. O comportamento é simplificado em relação ao NTP real.
O Packet Tracer não executa ataques reais. Cenários como DHCP falso são simulados com dispositivos adicionais, e não com ferramentas de ataque.
A disponibilidade de recursos (autenticação NTP, DHCP Snooping, entre outros) varia conforme a versão do Packet Tracer.
O comportamento do tempo no simulador pode exigir o uso do fast forward para sincronizações e convergência de portas.
14. Melhorias futuras
Melhoria	Objetivo	Projeto relacionado
VLAN de gerência dedicada	Tirar a gerência da VLAN 1	Segmentação com VLANs e ACLs
ACLs permitindo UDP 123 (NTP) e UDP 514 (Syslog) apenas entre origens e destinos corretos	Reduzir a superfície de ataque dos servidores	Segmentação com VLANs e ACLs
Autenticação NTP (ntp authenticate, chaves confiáveis)	Impedir um servidor de horário falso	Hardening de dispositivos
SSH em vez de Telnet e senhas fortes	Acesso administrativo seguro	Hardening de dispositivos
DHCP Snooping e port security	Bloquear servidor DHCP falso e dispositivos não autorizados	Segurança de camada 2
Segundo servidor DNS ou DHCP	Redundância	Projetos de alta disponibilidade
Análise dos logs com um SIEM (fora do Packet Tracer)	Correlação e detecção	Laboratórios com máquinas virtuais
15. Conceitos aprendidos
Segmentação de rede: separar clientes e servidores em redes distintas.
Roteamento entre redes: uso de um roteador com duas interfaces.
DHCP e DHCP relay: atribuição automática de endereços e encaminhamento de pedidos entre redes.
DNS: resolução de nomes e criação de registros do tipo A.
NTP: sincronização de tempo e hierarquia de stratum.
Syslog: centralização de logs e níveis de severidade.
Interface de gerência de switch: SVI na VLAN 1 e ip default-gateway.
Modos da CLI Cisco: usuário (>), privilegiado (#), configuração global ((config)#) e de interface ((config-if)#).
Métodos de validação: show ip interface brief, show ntp status, show logging, ping e nslookup.
Documentação técnica: registro de topologia, endereçamento, testes e limitações.

16. Referências

Materiais estudados durante a montagem do projeto:

Microsoft Learn: Dynamic Host Configuration Protocol (DHCP)
Cloudflare Learning Center: What is DNS?
PubNub: What is NTP?
Infoblox: What is IPAM?
Cisco: What is a LAN? e What Is a Wireless LAN?
Cloudflare Learning Center: What is a network switch? e What is a router?
18. Autor

Bruno, Estudante em transição para a área de cibersegurança.

Este projeto faz parte de uma série de laboratórios de redes com foco em cibersegurança. Sugestões e correções são bem-vindas.
