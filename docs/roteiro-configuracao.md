# Roteiro de configuração – CitizenTech (Windows Server)

Passo a passo da configuração do ambiente, feito pela interface gráfica. Todos os passos são no **SRV01**, salvo indicação contrária. Dois scripts de PowerShell, sem equivalente na interface, estão nos **Anexos A e B**.

**Ambiente:** SRV01 (Windows Server, `10.0.2.10`) e CLIENTE01 (Windows cliente, `10.0.2.20`). Domínio `corp.local`, NetBIOS `CORP`.

**Atalhos úteis** (tecla `Win + R`): `dnsmgmt.msc` (DNS), `dsa.msc` (Active Directory), `dfsmgmt.msc` (DFS), `gpmc.msc` (GPO), `inetmgr` (IIS), `perfmon` (Monitor de Desempenho), `taskschd.msc` (Agendador de Tarefas), `services.msc` (Serviços), `eventvwr` (Visualizador de Eventos), `wf.msc` (Firewall).

---

## 1. DNS

O DNS é instalado junto com o Active Directory e hospeda a zona `corp.local`.

1. Abra `dnsmgmt.msc` → *SRV01* → *Zonas de Pesquisa Direta* → `corp.local`.
2. Clique com o botão direito na zona → *Novo Alias (CNAME)*.
3. Em *Nome de alias*, digite `aquaguard`. Em *Nome de domínio totalmente qualificado do host de destino*, clique em *Procurar* e escolha o registro `SRV01`, ou digite `SRV01.corp.local`. Clique em *OK*.
4. No CLIENTE01, confira o DNS da placa de rede: `Win + R` → `ncpa.cpl` → botão direito na conexão → *Propriedades* → *Protocolo IP Versão 4* → *Propriedades*. O servidor DNS preferencial deve ser o IP do SRV01 (`10.0.2.10`).
5. **Teste:** no CLIENTE01, abra o navegador em `http://aquaguard.corp.local` (depois que o IIS estiver configurado, seção 6).

## 2. Active Directory

1. No *Gerenciador do Servidor* → *Gerenciar* → *Adicionar Funções e Recursos* → marque *Serviços de Domínio Active Directory*. Ao final, clique em *Promover este servidor a controlador de domínio*, escolha *Adicionar uma nova floresta* e use o nome `corp.local`.
2. Abra `dsa.msc`. Clique com o botão direito em `corp.local` → *Novo* → *Unidade Organizacional* e crie `Usuarios`, `Computadores`, `Grupos` e `GS`.
3. Dentro de `Usuarios`: botão direito → *Novo* → *Usuário* e crie os usuários de teste (por exemplo, `caio_RM560775`).
4. Dentro de `Grupos`: botão direito → *Novo* → *Grupo* e crie os grupos de segurança. Adicione os usuários aos grupos pelas *Propriedades* do grupo → aba *Membros*.
5. No CLIENTE01, ingresse no domínio: *Configurações* → *Sistema* → *Sobre* → *Domínio ou grupo de trabalho* → *Alterar* → *Domínio* → `corp.local`. Informe uma conta administradora do domínio e reinicie.
6. **Teste:** entre no CLIENTE01 com `CORP\<usuário>`. Em `dsa.msc`, o computador `CLIENTE01` aparece em *Computers* ou na OU `Computadores`.

## 3. File Server e permissões NTFS

1. No Explorador de Arquivos, crie em `E:\Dados` as pastas `Financeiro`, `projetos` e `Publico`.
2. **Compartilhar:** botão direito na pasta → *Propriedades* → aba *Compartilhamento* → *Compartilhamento Avançado* → marque *Compartilhar esta pasta* → *Permissões* → conceda *Controle Total* a `Todos` (o controle fino fica no NTFS).
3. **Permissões NTFS:** botão direito na pasta → *Propriedades* → aba *Segurança* → *Avançado*:
   1. Clique em *Desabilitar herança* → *Converter permissões herdadas em permissões explícitas*.
   2. Remova o grupo `Usuários` (se existir).
   3. *Editar* → *Adicionar* → escolha o grupo do setor e marque *Modificar* (leitura e gravação). Para um grupo somente de leitura, marque *Leitura e execução*.
   4. Mantenha `Administradores` e `SYSTEM` com *Controle total*.
4. **Teste:** no CLIENTE01, abra `\\SRV01\Publico` com um usuário que tem permissão (abre e grava) e com outro sem permissão (acesso negado).

## 4. DFS (namespace)

1. No *Gerenciador do Servidor* → *Adicionar Funções e Recursos* → *Serviços de Arquivo e Armazenamento* → *Serviços de Arquivo e iSCSI* → marque **Namespaces DFS**.
2. Abra `dfsmgmt.msc` → *Namespaces* (botão direito) → *Novo Namespace*.
3. Servidor: `SRV01`. Nome: `Arquivos`. Em *Tipo de Namespace*, escolha **Namespace baseado em domínio**. Conclua o assistente.
4. Dentro do namespace `\\corp.local\Arquivos`: botão direito → *Nova Pasta* → nome `Publico` → *Adicionar* → *Procurar* → escolha o compartilhamento `\\SRV01\Publico`.
5. **Teste:** no CLIENTE01, digite `\\corp.local\Arquivos\Publico` na barra de endereços do Explorador.

## 5. GPO

1. Abra `gpmc.msc` → *Floresta* → *Domínios* → `corp.local`.
2. Para criar uma GPO: botão direito em *Objetos de Política de Grupo* → *Novo* → informe o nome. Para vincular: botão direito na OU ou no domínio → *Vincular um GPO existente*. Para configurar: botão direito na GPO → *Editar*.

   | GPO | Vinculada a | Finalidade e configuração |
   |---|---|---|
   | `GPO-USR-Bloquear-Painel` | OU de usuários | *Configuração do Usuário* → *Modelos Administrativos* → *Painel de Controle* → **Proibir acesso ao Painel de Controle e às configurações do PC** = Habilitado |
   | `GPO-RDP-GS` | OU `GS` | *Configuração do Computador* → *Modelos Administrativos* → *Componentes do Windows* → *Serviços de Área de Trabalho Remota* → liberar conexões remotas |
   | `GPO-DFS` | Domínio ou OU de usuários | Acesso ao namespace DFS |
   | `GG-GPO-Piloto` | OU de teste | GPO piloto |

   *(Ajuste a tabela com a vinculação e a configuração reais de cada GPO.)*
3. **Aplicar no cliente:** faça logoff e entre de novo no CLIENTE01 (ou reinicie).
4. **Relatório:** botão direito na GPO → *Salvar Relatório* → formato HTML.
5. **Teste:** no CLIENTE01, a configuração da GPO deve estar valendo (por exemplo, o Painel de Controle fica bloqueado para o usuário da OU vinculada).

## 6. IIS (aplicação web)

**Instalação e arquivos**

1. *Gerenciador do Servidor* → *Adicionar Funções e Recursos* → marque **Servidor Web (IIS)**.
2. No Explorador de Arquivos, crie a pasta `C:\Sites\Aquaguard`. Coloque nela os arquivos `index.html` e `health.json`, que estão na pasta `web/` deste repositório.

**Pool de aplicativos**

3. Abra `inetmgr` → *Pools de Aplicativos* → botão direito → *Adicionar Pool de Aplicativos*.
4. Nome: `Aquaguard`. Versão do .NET CLR: **Sem Código Gerenciado**. *OK*.
5. Selecione o pool → *Configurações Avançadas* (painel direito) → *Modo de Início* = `AlwaysRunning`.

**Site**

6. *Sites* → botão direito → *Adicionar Site da Web*.
7. Nome do site: `Aquaguard`. Pool de aplicativos: `Aquaguard`. Caminho físico: `C:\Sites\Aquaguard`. Porta: `80`. Nome do host: `aquaguard.corp.local`. *OK*.

**Permissão da pasta**

8. Botão direito em `C:\Sites\Aquaguard` → *Propriedades* → *Segurança* → *Editar* → *Adicionar*. Em *Locais*, escolha o computador `SRV01`. Digite `IIS AppPool\Aquaguard` → *Verificar Nomes* → marque *Leitura e execução*.

**HTTPS (certificado autoassinado, para laboratório)**

9. Selecione o servidor `SRV01` (raiz da árvore do IIS) → *Certificados do Servidor* → *Criar Certificado Autoassinado* (painel direito) → nome `aquaguard.corp.local`, repositório *Pessoal*.
10. Selecione o site `Aquaguard` → *Associações* (painel direito) → *Adicionar*: tipo `https`, porta `443`, nome do host `aquaguard.corp.local`, marque *Exigir Indicação de Nome de Servidor* e escolha o certificado criado.

**Firewall**

11. `wf.msc` → *Regras de Entrada* → *Nova Regra* → *Porta* → *TCP* → portas específicas `80, 443` → *Permitir a conexão* → perfis de domínio, privado e público → nome `Aquaguard HTTP/HTTPS`.

**Reinício automático dos serviços**

12. `services.msc` → botão direito em **Serviço de Publicação da World Wide Web** → *Propriedades* → aba *Recuperação*. Defina *Primeira falha*, *Segunda falha* e *Falhas subsequentes* como **Reiniciar o Serviço**, e *Reiniciar o serviço após* = `1` minuto.
13. Repita o passo 12 para o serviço **Namespace DFS**.

**Teste:** no CLIENTE01, abra `http://aquaguard.corp.local` (use `Ctrl + F5`). O bloco "Status da plataforma" deve mostrar *Operacional*.

> **Atenção:** se o site retornar erro 500, verifique se existe um `web.config` na pasta do site com o tipo `.json` declarado sem `<remove fileExtension=".json" />`. O IIS já define esse tipo, e a declaração repetida gera erro. O site funciona sem `web.config`.

## 7. RDS (Serviços de Área de Trabalho Remota)

**Implantação**

1. *Gerenciador do Servidor* → *Gerenciar* → *Adicionar Funções e Recursos* → **Instalação dos Serviços de Área de Trabalho Remota**.
2. Escolha *Início Rápido* → *Implantação de área de trabalho baseada em sessão*.
3. Selecione o `SRV01` para o Agente de Conexão, o Acesso via Web e o Host da Sessão. Marque *Reiniciar o servidor de destino automaticamente* e clique em *Implantar*. O servidor reinicia.

**Coleção e aplicativo**

4. Depois do reinício, abra o *Gerenciador do Servidor* → *Serviços de Área de Trabalho Remota* → *Coleções* → *Tarefas* → *Criar Coleção de Sessões*.
5. Nome: `CT-Aquaguard`. Servidor: `SRV01`. Em *Grupos de usuários*, defina quem pode acessar (por exemplo, o grupo do setor).
6. Em *Coleção* → *Programas RemoteApp* → *Tarefas* → *Publicar Programas RemoteApp*. Escolha **Microsoft Edge** (ou *Adicionar* e procure `msedge.exe`).
7. Clique com o botão direito no programa publicado → *Editar Propriedades* → *Parâmetros* → *Sempre usar os seguintes parâmetros de linha de comando* → `http://aquaguard.corp.local`.
8. **Teste:** a coleção `CT-Aquaguard` aparece na lista de coleções, com o `SRV01` como host.

> O SRV01 também é controlador de domínio. Isso funciona em laboratório, mas não é recomendado em produção.

## 8. Monitoramento (Monitor de Desempenho)

São três alertas. Repita os passos para cada um, mudando nome, contador e limite:

| Nome | Contador | Condição | Limite |
|---|---|---|---|
| `CT_Alerta_CPU` | `\Processor(_Total)\% Processor Time` | Acima de | 85 |
| `CT_Alerta_Memoria` | `\Memory\Available MBytes` | Abaixo de | 512 |
| `CT_Alerta_Disco` | `\LogicalDisk(C:)\% Free Space` | Abaixo de | 20 |

1. Abra `perfmon` → *Conjuntos de Coletores de Dados* → *Definido pelo Usuário* → botão direito → *Novo* → *Conjunto de Coletores de Dados*.
2. Digite o nome do alerta e escolha **Criar manualmente (Avançado)** → *Avançar*.
3. Marque **Alerta do Contador de Desempenho** → *Avançar*.
4. Em *Intervalo de amostragem*, digite `10` segundos. Clique em *Adicionar*, escolha o contador da tabela, defina *Acima de* ou *Abaixo de* conforme a coluna *Condição* e digite o limite. *Concluir*.
5. Dentro do conjunto criado, dê dois cliques no coletor (ícone de alerta) → aba **Ação de Alerta** → marque **Registrar uma entrada no log de eventos do aplicativo** → *OK*.
6. Botão direito no conjunto → **Iniciar**. O status passa a *Em execução*.

**Iniciar os alertas junto com o servidor**

7. `taskschd.msc` → *Criar Tarefa Básica*. Nome: `Iniciar_CT_Alerta_CPU`. Gatilho: *Quando o computador iniciar*. Ação: *Iniciar um programa*. Programa: `logman`. Argumentos: `start CT_Alerta_CPU`. Em *Propriedades* da tarefa, marque *Executar estando o usuário conectado ou não* e *Executar com privilégios mais altos*, usando a conta `SYSTEM`. Repita para os outros dois alertas.

**Verificador complementar**

8. Crie a pasta `C:\CitizenTech` e salve nela o arquivo `monitor-recursos.ps1` com o conteúdo do **Anexo A** (Bloco de Notas → *Salvar como* → tipo *Todos os arquivos* → nome `monitor-recursos.ps1`).
9. No Agendador de Tarefas, crie a tarefa `CT-MonitorRecursos`, conforme o **item 9.2** (mesmo procedimento da `CT-HealthCheck`), apontando para esse arquivo.

Esse verificador grava eventos no log *Aplicativo* (fonte `CitizenTechMonitor`): **4001** CPU, **4002** memória, **4003** disco e **4000** (tudo normal).

**Teste:** o Monitor de Desempenho lista os três alertas em *Em execução*. No Visualizador de Eventos (`eventvwr` → *Logs do Windows* → *Aplicativo*), os eventos **4000** aparecem a cada minuto.

## 9. Recuperação automática de falhas

A tarefa `CT-HealthCheck` verifica o site e o DFS a cada minuto e religa o que estiver parado. Os eventos ficam no log *Aplicativo* (fonte `CitizenTechHealth`): **5001** falha detectada, **5002** recuperado e **5003** recuperação falhou.

**9.1 Script**

1. Salve o conteúdo do **Anexo B** em `C:\CitizenTech\healthcheck.ps1` (Bloco de Notas, tipo *Todos os arquivos*).

**9.2 Tarefa agendada** (o mesmo procedimento vale para a `CT-MonitorRecursos`, trocando o nome e o arquivo)

2. `taskschd.msc` → *Criar Tarefa* (não a básica).
3. Aba **Geral:** nome `CT-HealthCheck`. Clique em *Alterar Usuário ou Grupo*, digite `SYSTEM` e confirme. Marque *Executar estando o usuário conectado ou não* e *Executar com privilégios mais altos*.
4. Aba **Gatilhos:** *Novo* → *Iniciar a tarefa:* **Em um agendamento** → *Diariamente*. Marque *Repetir a tarefa a cada:* `1 minuto`, *por uma duração de:* **Indefinidamente**. Marque *Habilitado* → *OK*.
5. Aba **Ações:** *Novo* → *Iniciar um programa*. Programa/script: `powershell.exe`. Adicionar argumentos: `-NoProfile -ExecutionPolicy Bypass -File C:\CitizenTech\healthcheck.ps1`. *OK*.
6. Clique em *OK* para criar a tarefa. **Teste:** a tarefa aparece com status *Pronto*.

## 10. Teste de falha

**Falha do IIS**

1. No CLIENTE01, abra `http://aquaguard.corp.local` e confirme que a página carrega.
2. No SRV01, abra `inetmgr` → *Sites* → `Aquaguard` → painel direito → *Gerenciar Site da Web* → **Parar**. O ícone mostra o site parado.
3. No CLIENTE01, atualize a página (`F5`): aparece erro de conexão.
4. Aguarde até **1 minuto**. No `inetmgr`, atualize (`F5`): o site volta a *Iniciado*. No CLIENTE01, a página volta a carregar.

**Falha do DFS**

5. No SRV01, abra `services.msc`, localize **Namespace DFS** (status *Em execução*), botão direito → **Parar**.
6. Aguarde até **1 minuto** e atualize a lista (`F5`): o serviço volta para *Em execução*.
7. No CLIENTE01, confirme que `\\corp.local\Arquivos\Publico` abre.

**Eventos de recuperação**

8. No SRV01, abra `eventvwr` → *Logs do Windows* → *Aplicativo* → painel direito → *Filtrar Log Atual…* → em *Origens do evento*, marque **CitizenTechHealth** → *OK*. Devem aparecer os pares **5001** (falha detectada) e **5002** (recuperado), um para o IIS e um para o DFS.

**Disparo do alerta de disco, sem sobrecarregar a VM**

9. Abra `C:\CitizenTech\monitor-recursos.ps1` no Bloco de Notas e troque, na linha do disco, `$dsk -lt 20` por `$dsk -lt 99`. Salve.
10. Aguarde até **2 minutos** (a tarefa roda a cada minuto). No Visualizador de Eventos, filtre por **ID 4003**: aparece um evento de nível *Aviso*, "ALERTA Disco C:…".
11. **Volte** `$dsk -lt 99` para `$dsk -lt 20` e salve.

## 11. Evidências a coletar

| Pasta em `evidencias/` | O que registrar (prints) |
|---|---|
| `01-ad-dns` | `dnsmgmt.msc` com a zona `corp.local` e o registro `aquaguard`; `dsa.msc` com as OUs, usuários e o computador CLIENTE01 |
| `02-fileserver-ntfs` | Compartilhamentos (*Gerenciamento do Computador* → *Pastas Compartilhadas*); aba *Segurança* de uma pasta |
| `03-dfs` | `dfsmgmt.msc` com o namespace `\\corp.local\Arquivos`; `\\corp.local\Arquivos\Publico` aberto no cliente |
| `04-iis` | `inetmgr` com o site e o pool; as associações (*Associações*); o site aberto no CLIENTE01 |
| `05-rds` | *Gerenciador do Servidor* → *Serviços de Área de Trabalho Remota* → coleção `CT-Aquaguard` |
| `06-gpo` | `gpmc.msc` com as GPOs e as vinculações; relatório HTML salvo |
| `07-monitoramento` | `perfmon` com os três alertas *Em execução*; evento 4003 no Visualizador de Eventos |
| `08-teste-falha` | Site parado e depois iniciado; serviço DFS parado e depois em execução; eventos 5001 e 5002; Agendador de Tarefas com `CT-HealthCheck` e `CT-MonitorRecursos` |

---

## Anexo A – `monitor-recursos.ps1`

Salve em `C:\CitizenTech\monitor-recursos.ps1`.

```powershell
if (-not [System.Diagnostics.EventLog]::SourceExists("CitizenTechMonitor")) { New-EventLog -LogName Application -Source "CitizenTechMonitor" }
$cpu=(Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples.CookedValue
$mem=(Get-Counter '\Memory\Available MBytes').CounterSamples.CookedValue
$dsk=(Get-Counter '\LogicalDisk(C:)\% Free Space').CounterSamples.CookedValue
$ok=$true
if ($cpu -gt 85){ Write-EventLog -LogName Application -Source CitizenTechMonitor -EventId 4001 -EntryType Warning -Message ("ALERTA CPU alta: {0:N1}% (limite 85%)" -f $cpu); $ok=$false }
if ($mem -lt 512){ Write-EventLog -LogName Application -Source CitizenTechMonitor -EventId 4002 -EntryType Warning -Message ("ALERTA Memoria baixa: {0:N0} MB livres (limite 512 MB)" -f $mem); $ok=$false }
if ($dsk -lt 20){ Write-EventLog -LogName Application -Source CitizenTechMonitor -EventId 4003 -EntryType Warning -Message ("ALERTA Disco C: com {0:N1}% livre (limite 20%)" -f $dsk); $ok=$false }
if ($ok){ Write-EventLog -LogName Application -Source CitizenTechMonitor -EventId 4000 -EntryType Information -Message ("OK CPU {0:N0}% | Mem livre {1:N0} MB | Disco livre {2:N0}%" -f $cpu,$mem,$dsk) }
```

## Anexo B – `healthcheck.ps1`

Salve em `C:\CitizenTech\healthcheck.ps1`.

```powershell
Import-Module WebAdministration
if (-not [System.Diagnostics.EventLog]::SourceExists('CitizenTechHealth')) { New-EventLog -LogName Application -Source 'CitizenTechHealth' }
function Testar {
  $erros = @()
  try { $r = Invoke-WebRequest 'http://localhost/health.json' -Headers @{Host='aquaguard.corp.local'} -UseBasicParsing -TimeoutSec 5
        if ($r.StatusCode -ne 200) { $erros += "IIS HTTP $($r.StatusCode)" } } catch { $erros += "IIS: $_" }
  if (-not (Test-Path '\\corp.local\Arquivos\Publico')) { $erros += 'DFS: namespace inacessivel' }
  if ((Get-Service DFS).Status -ne 'Running') { $erros += 'Servico DFS parado' }
  return $erros
}
$e = Testar
if ($e.Count -gt 0) {
  Write-EventLog -LogName Application -Source CitizenTechHealth -EventId 5001 -EntryType Error -Message ('Falha detectada: ' + ($e -join ' | ') + '. Iniciando recuperacao.')
  foreach ($s in 'W3SVC','DFS','LanmanServer') { if ((Get-Service $s).Status -ne 'Running') { Start-Service $s } }
  if ((Get-WebAppPoolState Aquaguard).Value -ne 'Started') { Start-WebAppPool Aquaguard }
  if ((Get-WebsiteState Aquaguard).Value -ne 'Started') { Start-Website Aquaguard }
  Start-Sleep 8
  $e2 = Testar
  if ($e2.Count -eq 0) { Write-EventLog -LogName Application -Source CitizenTechHealth -EventId 5002 -EntryType Information -Message 'Servicos recuperados automaticamente.' }
  else { Write-EventLog -LogName Application -Source CitizenTechHealth -EventId 5003 -EntryType Error -Message ('Recuperacao falhou: ' + ($e2 -join ' | ')) }
}
```
