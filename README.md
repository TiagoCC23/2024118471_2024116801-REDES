# Redes de Computadores I — Plano do Projeto (reinício)

**UFP · Docente:** Pedro Sobral · **Ano letivo:** 2026/2027
**Repositório:** `2024118471_2024116801-RDS1`

Este documento reorganiza o enunciado em **duas fases** e serve de base às issues (`ISSUES.md`).

---

## 0. Ponto de partida

Recomeço **do zero**: nada do trabalho anterior é reaproveitado (projeto GNS3, configs e checklist do README são refeitos). Dois cuidados desde o início:

- **Guardar sempre** (`write memory`) e fazer commit no fim de cada issue.
- **Confirmar cedo** que a imagem usada em CD1/CD2 suporta `ip routing`, SVIs, OSPF e **VRRP** (ver risco R1).

---

## 1. Topologia (mapa de ligações extraído do `.gns3`)

| Ligação | Extremo A | Extremo B | Tipo |
|---|---|---|---|
| CD1 ↔ Edge | CD1 e0/0 | Edge e0/0 | L3 `10.50.15.0/30` |
| CD2 ↔ Edge | CD2 e0/0 | Edge e0/1 | L3 `10.50.15.4/30` |
| Edge ↔ Internet | Edge e0/2 | nat0 | NAT outside |
| CD1 ↔ CD2 | CD1 e0/1, e0/2 | CD2 e0/1, e0/2 | L3 `10.50.15.8/30` (Port-Channel routed, LACP) |
| CD1 ↔ A1 | CD1 e1/0, e1/1 | A1 e0/0, e0/1 | Trunk, LACP |
| CD1 ↔ A2 | CD1 e2/0, e2/1 | A2 e0/0, e0/1 | Trunk, LACP |
| CD1 ↔ A3 | CD1 e3/0, e3/1 | A3 e0/0, e0/1 | Trunk, LACP |
| CD2 ↔ A1 | CD2 e1/0, e1/1 | A1 e0/2, e0/3 | Trunk, LACP |
| CD2 ↔ A2 | CD2 e2/0, e2/1 | A2 e0/2, e0/3 | Trunk, LACP |
| CD2 ↔ A3 | CD2 e3/0, e3/1 | A3 e0/2, e0/3 | Trunk, LACP |
| Server ↔ CD1 | ens4 | CD1 e0/3 | Trunk 802.1Q (bonding) |
| Server ↔ CD2 | ens5 | CD2 e0/3 | Trunk 802.1Q (bonding) |

**Portas de acesso**

| Switch | Porta | Dispositivo | VLAN |
|---|---|---|---|
| A1 | e1/0 | PC1 | 10 |
| A1 | e1/1 | PC4 | 10 |
| A2 | e1/0 | PC2 | 20 |
| A2 | e1/1 | PC5 | 20 |
| A3 | e1/0 | PC3 (Attacker) | 30 |
| A3 | e1/1 | Management PC | 99 |

> Nota: o enunciado coloca PC4 na VLAN 10 e PC5 na VLAN 20; a figura mostra os pares em A1 e A2.

**Port-Channels propostos**

| Switch | Po | Membros | Destino |
|---|---|---|---|
| A1/A2/A3 | Po1 | e0/0, e0/1 | CD1 |
| A1/A2/A3 | Po2 | e0/2, e0/3 | CD2 |
| CD1 e CD2 | Po1 / Po2 / Po3 | pares para A1 / A2 / A3 | Acesso |
| CD1 e CD2 | Po12 | e0/1, e0/2 | Entre CD (routed, Fase 2) |

---

## 2. Endereçamento VLSM (bloco `10.50.0.0/20`)

Alocar do maior para o menor.

### Opção A — dimensionamento literal do enunciado

| Rede | Hosts req. | Máscara | Sub-rede | Gama utilizável | Broadcast |
|---|---:|---|---|---|---|
| VLAN 20 Student | 120 | /25 (255.255.255.128) | 10.50.0.0 | .1 – .126 | 10.50.0.127 |
| VLAN 10 Guest | 60 | /26 (255.255.255.192) | 10.50.0.128 | .129 – .190 | 10.50.0.191 |
| VLAN 30 Staff/Server | 30 | /27 (255.255.255.224) | 10.50.0.192 | .193 – .222 | 10.50.0.223 |
| VLAN 99 Management | 14 | /28 (255.255.255.240) | 10.50.0.224 | .225 – .238 | 10.50.0.239 |
| CD1–Edge | — | /30 | 10.50.15.0 | .1 – .2 | .3 |
| CD2–Edge | — | /30 | 10.50.15.4 | .5 – .6 | .7 |
| CD1–CD2 | — | /30 | 10.50.15.8 | .9 – .10 | .11 |

### ⚠️ Problema a decidir
60, 30 e 14 são exatamente o máximo utilizável de /26, /27 e /28. Mas cada VLAN também precisa de **3 IPs de gateway** (VIP VRRP + SVI do CD1 + SVI do CD2). Com a Opção A, essas VLANs ficam **sem folga** (ex.: Guest só comporta 59 clientes).

### Opção B — com margem para os gateways (recomendada, a confirmar com o professor)

| Rede | Máscara | Sub-rede | Gama utilizável |
|---|---|---|---|
| VLAN 20 Student | /25 | 10.50.0.0 | .1 – .126 |
| VLAN 10 Guest | /25 | 10.50.0.128 | .129 – .254 |
| VLAN 30 Staff/Server | /26 | 10.50.1.0 | .1 – .62 |
| VLAN 99 Management | /27 | 10.50.1.64 | .65 – .94 |
| P2P (3×) | /30 | 10.50.15.0 / .4 / .8 | conforme enunciado |

**Convenção de gateways (ambas as opções):** VIP = primeiro IP utilizável, CD1 = segundo, CD2 = terceiro. Server (VLAN 30) com IP fixo no final da gama. Loopbacks para Router-ID: `10.50.14.0/24` (ex.: Edge 1.1, CD1 .2, CD2 .3 com /32) — **a decidir e documentar**.

---

## 3. FASE 1 — Infraestrutura, Layer 2 e planeamento IP

**Objetivo:** topologia montada e estável, L2 totalmente funcional e validado, endereçamento calculado e documentado. Corresponde ao **Objetivo 1** do enunciado + cálculo VLSM do Objetivo 2.

### Entregas da Fase 1
1. Projeto GNS3 limpo, com todos os nós renomeados e ligados como na secção 1.
2. VLANs 10, 20, 30, 99 em A1–A3, CD1, CD2.
3. Trunks 802.1Q com `switchport mode trunk` + `switchport nonegotiate`.
4. 6 EtherChannels LACP de acesso (+ Po12 CD1–CD2 criado e validado em L2/L3 na Fase 2).
5. Rapid-PVST+ com roots: CD1 primary 10/30 e secondary 20/99; CD2 primary 20/99 e secondary 10/30.
6. PortFast + BPDU Guard nas portas de acesso de A1–A3.
7. Bonding `active-backup` (bond0) no Server com VLAN 30 sobre trunk, e teste de failover.
8. Tabela VLSM final (A ou B), validada sem sobreposições.
9. Configs guardadas + evidências + README atualizado.

### Testes de validação (guardar prints em `documentation/evidence/fase1/`)
`show vlan brief` · `show interfaces trunk` · `show etherchannel summary` · `show spanning-tree vlan X` · `show spanning-tree summary` · `show interfaces status` · `cat /proc/net/bonding/bond0`

---

## 4. FASE 2 — Layer 3, serviços, NAT/ACL e segurança

**Objetivo:** tudo o que depende da base L2: gateway redundante, routing, serviços no Debian, saída para a Internet, políticas de acesso e ataques/contramedidas. Corresponde aos **Objetivos 2 a 7**.

### Blocos da Fase 2 (ordem recomendada)

| Bloco | Conteúdo | Obj. |
|---|---|---|
| 2.1 | SVIs, `ip routing`, VRRP (prioridades por VLAN), tracking | 2 |
| 2.2 | Links L3 routed CD1/CD2/Edge, Loopback0, OSPFv2 Área 0, passive-interface, default-information originate | 3 |
| 2.3 | Edge: NAT/PAT overload + Static NAT do Web | 4 |
| 2.4 | Server: isc-dhcp-server (pools 10/20/30), `ip helper-address` nas SVIs | 2 |
| 2.5 | Server: BIND9 (org.local, zona direta/inversa, forwarders, allow-query) | 6 |
| 2.6 | ACLs: gestão (VLAN 99 + Management PC), Guest, proteção do Server | 5 |
| 2.7 | Simulação de falhas: VRRP, OSPF, bonding | 2/3 |
| 2.8 | Ataques Yersinia/Scapy + contramedidas: DHCP Snooping, DAI, IP Source Guard, VRRP auth, BPDU Guard, Root Guard, nonegotiate | 7 |
| 2.9 | Capturas Wireshark (DNS, VRRP, OSPF LSA, DHCP, BPDU, NAT) | 3/4/7 |
| 2.10 | Relatório técnico, exportação do projeto GNS3, preparação da defesa | 5 |

---

## 5. Mapeamento com os critérios de avaliação

| Nível | Requisitos | Onde fica |
|---|---|---|
| Suficiente (10–13) | VLSM, L2/L3 base, OSPF + PAT, DHCP + BIND9 básico + helper, VRRP, bonding | Fase 1 inteira + 2.1–2.5 |
| Bom (14–17) | VRRP alinhado com STP root, ACLs estendidas (incl. DNS), simulação de falhas, Yersinia + DHCP Snooping + BPDU Guard | 2.6, 2.7, 2.8 (básico) |
| Muito Bom (18–20) | BIND9 completo, Static NAT, DAI, auth VRRP, 3 ataques, análise Wireshark, 90% dos conceitos | 2.5, 2.3, 2.8 (completo), 2.9 |

---

## 6. Convenções do repositório

- Branch por issue: `fase1/vlans`, `fase2/ospf`… e Pull Request para `main`.
- Commits curtos e frequentes: `feat(L2): configura trunks CD1↔A1 (#4)`.
- Configs exportadas para `configs/<equipamento>.cfg` após cada issue (`show running-config` / `write memory`).
- Evidências em `documentation/evidence/faseN/<issue>-descricao.png`.
- Não versionar `private-config.cfg`, discos qemu/docker, `.pcap`.
- Cada issue só fecha com **evidência anexada**.

---

## 7. Riscos

| # | Risco | Mitigação |
|---|---|---|
| R1 | Imagem IOU L2 pode não suportar VRRP/OSPF/routed ports em CD1/CD2 | Testar na issue de arranque; se falhar, trocar CD1/CD2 por imagem L3 (`x86_64_crb_linux-...`) |
| R2 | RAM/CPU da VM (IOU 1024 MB × 6 + Debian + Docker) | Ligar só o necessário; verificar recursos antes da Fase 2 |
| R3 | Dimensionamento VLSM ambíguo | Decidir A/B na primeira aula e documentar |
| R4 | Tempo até à defesa | Fase 1 primeiro e fechada; só depois Fase 2 |
