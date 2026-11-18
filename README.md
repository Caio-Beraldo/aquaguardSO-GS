# CitizenTech – Infraestrutura Windows Server

Infraestrutura Windows Server que hospeda a aplicação web da solução **CitizenTech** (plataforma de alertas de enchentes, projeto AlertaCheia Brasil). O ambiente aplica os serviços do semestre (DNS, Active Directory, File Server, DFS, IIS, GPO e RDS), é monitorado pelo Monitor de Desempenho e se recupera automaticamente de falhas.

## 1. Identificação

| Item | Informação |
|---|---|
| Projeto | CitizenTech / Aquaguard |
| Equipe | [Caio Costa Beraldo – RM560775] [Victor Kenzo Mikado – RM560057] [Ícaro Henrique de Souza Calixto – RM560278] [Henrique Celso Moura da Paixão Filho – RM559687] |
| Turma | [3ESPA] |
| Vídeo | [https://youtu.be/UubBrO97RIg] |

## 2. Ambiente

| VM | Sistema | Funções |
|---|---|---|
| **SRV01** | Windows Server | AD DS, DNS, DHCP, File Server, DFS Namespace, IIS e RDS |
| **CLIENTE01** | Windows cliente | Ingressado no domínio; usado para testar o acesso |

Domínio: `corp.local` (NetBIOS `CORP`). 

## 3. Serviços implantados

| Serviço | O que foi configurado | Evidências |
|---|---|---|
| **DNS** | Zona `corp.local` e registro `aquaguard` apontando para o SRV01 | [`01-ad-dns`](evidencias/01-ad-dns) |
| **Active Directory** | OUs `Usuarios`, `Computadores`, `Grupos` e `GS`; usuários e computador cliente ingressados no domínio | [`01-ad-dns`](evidencias/01-ad-dns) |
| **File Server** | `E:\Dados` com os compartilhamentos `Financeiro`, `projetos` e `Publico`; permissões NTFS por grupo | [`02-fileserver-ntfs`](evidencias/02-fileserver-ntfs) |
| **DFS** | Namespace de domínio `\\corp.local\Arquivos` com a pasta `Publico` | [`03-dfs`](evidencias/03-dfs) |
| **IIS** | Site `aquaguard` (`http://aquaguard.corp.local`), pool de aplicação próprio, HTTPS e endpoint `/health.json` | [`04-iis`](evidencias/04-iis) |
| **RDS** | Implantação em servidor único, coleção `CT-Aquaguard` e RemoteApp | [`05-rds`](evidencias/05-rds) |
| **GPO** | `GPO-USR-Bloquear-Painel`, `GPO-RDP-GS`, `GPO-DFS` e `GG-GPO-Piloto` | [`06-gpo`](evidencias/06-gpo) |
| **Monitoramento** | Alertas de CPU, memória e disco no Monitor de Desempenho | [`07-monitoramento`](evidencias/07-monitoramento) |
| **Resposta a falhas** | Verificação a cada minuto com recuperação automática do IIS e do DFS | [`08-teste-falha`](evidencias/08-teste-falha) |

## 4. Roteiro de configuração

O passo a passo completo, feito pela interface gráfica, está em **[`docs/roteiro-configuracao.md`](docs/roteiro-configuracao.md)**. Ele cobre, na ordem:

1. DNS
2. Active Directory
3. File Server e permissões NTFS
4. DFS (namespace)
5. GPO
6. IIS (aplicação web, HTTPS, firewall)
7. RDS
8. Monitoramento (Monitor de Desempenho)
9. Recuperação automática de falhas
10. Teste de falha
11. Evidências a coletar

Os anexos do roteiro trazem os dois scripts de PowerShell usados pelas tarefas agendadas (verificador de recursos e health check).

## 5. Monitoramento

Alertas criados no Monitor de Desempenho (Conjuntos de Coletores de Dados → Definido pelo Usuário):

| Alerta | Condição |
|---|---|
| `CT_Alerta_CPU` | CPU acima de 85% |
| `CT_Alerta_Memoria` | Menos de 512 MB de memória disponível |
| `CT_Alerta_Disco` | Menos de 20% de espaço livre em C: |

Uma tarefa agendada (`CT-MonitorRecursos`, script no Anexo A do roteiro) complementa os alertas e grava eventos no log Aplicativo (fonte `CitizenTechMonitor`): **4001** CPU, **4002** memória, **4003** disco e **4000** (tudo normal).

## 6. Teste de falha

Vídeo: [https://youtu.be/UubBrO97RIg]. Duas falhas são provocadas, e o ambiente se recupera sozinho:

| Falha | Como é provocada | Recuperação |
|---|---|---|
| **IIS** | Parar o site `aquaguard` no Gerenciador do IIS | A tarefa `CT-HealthCheck` (roteiro, seção 9 e Anexo B) detecta e religa o site em até 1 minuto |
| **DFS** | Parar o serviço *Namespace DFS* em `services.msc` | A mesma tarefa religa o serviço em até 1 minuto |

O health check registra no log Aplicativo (fonte `CitizenTechHealth`) o evento **5001** (falha detectada) e o **5002** (recuperado). Também são mostrados o alerta de disco (evento 4003) e os alertas em execução.

## 7. Evidências

Cada pasta de `evidencias/` contém prints das ferramentas de administração.

| Pasta | Conteúdo |
|---|---|
| `01-ad-dns` | DNS Manager e Usuários e Computadores do AD |
| `02-fileserver-ntfs` | Compartilhamentos e permissões NTFS |
| `03-dfs` | Namespace `\\corp.local\Arquivos` |
| `04-iis` | Gerenciador do IIS e o site no cliente |
| `05-rds` | Coleção `CT-Aquaguard` |
| `06-gpo` | GPMC, lista de GPOs e relatório |
| `07-monitoramento` | Alertas em execução e evento 4003 |
| `08-teste-falha` | Falhas, eventos 5001 e 5002, tarefas agendadas |

## 8. Limitações

- Um único servidor concentra todos os serviços, que é um ponto único de falha.
- O DFS tem um servidor de namespace e não há replicação (DFS-R).
- O RDS roda no controlador de domínio, o que é aceitável em laboratório e não é recomendado em produção.
- Certificado HTTPS autoassinado e usuários de teste.

## 9. Estrutura do repositório

```
README.md
web/          index.html, health.json, web.config (aplicação web)
docs/         roteiro de configuração
evidencias/   prints das ferramentas (01 a 08)
```
