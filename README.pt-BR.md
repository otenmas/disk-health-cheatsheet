<p align="right"><a href="README.md">🇺🇸 English</a></p>

# disk-health-cheatsheet

Referência rápida para verificar a saúde e a integridade de HDDs e SSDs usando comandos de terminal e PowerShell. O foco é a verificação de discos sobressalentes, pera avaliar sua condicao de recuperacao, ou reaproveitamento apos testes e limpeza. Apesar do foco nao ser a integridade dos discos do sistema operacional, foi implantado uma secao para esse fim. Os comandos foram testados com o Windows 11, utilizando o PowerShell, rodando como administrador. no CMD alguns comandos nao funcionam, precisa verificar os comandos correlatos. Tambem foi cirada uma secao para o Linux.

## Índice

- [Antes de começar](#antes-de-começar)
- [Fluxo de trabalho de validação (passo a passo)](#fluxo-de-trabalho-de-validação-passo-a-passo)
- [Windows (PowerShell / CMD)](#windows-powershell--cmd)
- [Como interpretar os dados SMART](#como-interpretar-os-dados-smart)
- [Sinais de alerta](#sinais-de-alerta)
- [Limpeza, formatação e criação de partição](#limpeza-formatação-e-criação-de-partição)
- [Integridade dos arquivos do Windows](#Integridade_dos_arquivos_do_Windows)
- [Linux (terminal)](#linux-terminal)
- [Licença](#licença)

## Antes de começar

- A maioria dos comandos exige **administrador** (Windows) ou **root/sudo** (Linux).
- **Faça backup dos seus dados primeiro** se o disco já apresenta problemas. Ferramentas de reparo podem piorar um disco com defeito.
- Comandos marcados como *somente leitura* não alteram nada no disco.
- Substitua `/dev/sdX`, `/dev/nvme0` e `C:` pelo seu dispositivo ou letra de unidade.
- ⚠️ Comandos com `C:` atuam no disco **do próprio computador**, e não no disco conectado ao dock.

## Fluxo de trabalho de validação (passo a passo)

Roteiro para validar discos SATA descartados (ou novos). Normalmente Tecnicos de Informatica, funcionarios de TI, que trabalham com Hardware, e constantemente trabalham com discos (HDD, SSD), utilizam dock station para HD/SSD USB para testar e clonar os discos. Por isso algums parametros são necessarios nos comandos do **SMART**. No caso de testar os discos conectados diretamente em porta SATA da placa mãe, que em geral é **mais confiável**, os parametros podem mudar.

| Ponto	| Dock / case USB |	SATA direto na placa-mãe |
|-------|-----------------|--------------------------|
| **Comando SMART** |	Precisa do `-d sat` (e às vezes `sat,12`, `usbjmicron`, `usbsunplus`) |	Normalmente sem `-d`: smartctl -a /dev/sdX |
| **SMART passa?** |	Depende do chip do dock. Alguns bloqueiam |	Sempre passa, desde que a BIOS esteja em modo **AHCI** |
| `BusType` |	`USB` |	`SATA` (ou `RAID`, se estiver em modo RAID/Intel RST) |
| **Serial no Windows** |	Genérico (ex.: `1234567890123`) |	Serial real do disco |
| `MediaType` |	Quase sempre `Unspecified` |	Mostra `SSD` ou `HDD` corretamente |
| `Get-StorageReliabilityCounter` |	Costuma vir vazio |	Costuma funcionar (temperatura, desgaste, horas ligado) |
| **Velocidade (CrystalDiskMark)** |	Limitada pelo dock e pela porta USB |	Velocidade real do disco (SATA III, até ~550 MB/s) |
| **Testes longos** |	Risco de o dock desligar ou a USB suspender |	Mais estável |
| **Conexão** |	Hot-swap, o dock já fornece energia |	Precisa de cabo SATA e cabo de energia |
| **Suspeita de CRC alto** |	Dock, cabo USB ou porta |	Cabo SATA ou porta |

Teste **um disco por vez**.

**Pontos de atenção para a conexão direta:**

- **Desligue o PC antes de conectar o disco, a menos que tenha certeza de que o hot-plug está habilitado na BIOS. Isso evita danos e travamentos.
- **Ordem de boot:** se o disco tiver um sistema operacional, o PC pode tentar iniciar por ele. Confira a ordem de boot na BIOS se o PC não iniciar normalmente.
- **Risco de apagar o disco errado é maior**, pois o disco fica dentro do computador ao lado do disco do sistema. A checagem de `IsBoot`/`IsSystem` antes do `Clear-Disk` é ainda mais importante.
- **Modo RAID/Intel RST:** em alguns notebooks e PCs de empresa, a BIOS vem em modo RAID e o Windows pode esconder o SMART ou não enxergar o disco. Se ocorrer, mude para AHCI (com cuidado: mexer nisso no disco do sistema pode impedir o Windows de iniciar).
- **Windows pode montar o disco automaticamente** se ele tiver partições conhecidas (NTFS, por exemplo). Isso só importa se o disco tiver letra: nesse caso, os comandos de sistema de arquivos passam a se aplicar, embora, como você vai formatar depois, não sejam necessários.

**Pontos de atenção para dock/case:**

- Docks baratos podem não repassar o SMART, mostrar capacidade errada ou ter limite de tamanho em chips muito antigos.
- Para o desempenho, vale ter um dock com **UASP** e conectar em uma porta USB 3.x (azul ou USB-C). Em USB 2.0 a velocidade cai para algo em torno de 35 a 40 MB/s.
- Um case (disco fechado dentro de uma caixa) se comporta como um dock, mas esquenta mais, então vale ficar de olho na temperatura durante o teste longo.

### Passo 1: Preparação

- Etiquete o disco e anote modelo, serial e capacidade.
- Abra o PowerShell **como Administrador**.
- Instale o smartmontools (veja [Instalar o smartmontools](#instalar-o-smartmontools)).
- Desative a suspensão do PC e a suspensão seletiva de USB, para não interromper os testes longos.

### Passo 2: Identificação

```powershell
Get-Disk
smartctl --scan
smartctl -i /dev/sdX -d sat
```

- Troque o sdX pela **id** do disco. (sda para o disco 0, sdb para o 1, sdc para o 2, etc.)
- Confirme no `Get-Disk` que o `BusType` é **USB** e anote o número do disco.
- Confirme **modelo e serial reais** no `smartctl -i` (compare com a etiqueta). O serial mostrado pelo Windows costuma ser genérico quando o disco está no dock.
- Verifique se aparece `SMART support is: Available` e `Enabled`.
- Se o Windows pedir para formatar o disco RAW, clique em **Cancelar**.

### Passo 3: Análise passiva (SMART)

```powershell
smartctl -a /dev/sdX -d sat
```

**Anote os valores iniciais** e verifique:

- `overall-health`: deve ser **PASSED**
- `SSD_Life_Left` (vida útil restante, em SSDs)
- `Reallocated_Event_Count` ou `Reallocated_Sector_Ct`: ideal **0**
- `Current_Pending_Sector` e `Offline_Uncorrectable`: devem ser **0**
- `CRC_Error_Count`: se alto, suspeite do cabo ou do dock

### Passo 4: Testes ativos (autotestes SMART)

```powershell
smartctl -t short /dev/sdX -d sat
smartctl -l selftest /dev/sdX -d sat

smartctl -t long /dev/sdX -d sat
smartctl -l selftest /dev/sdX -d sat
```

- A primeira linha é para rodar o teste, aguarde o tempo correspondente, e verifique o resultado utilizando a segunda linha.
- O teste curto leva cerca de 2 minutos. O longo pode levar horas.
- Não desconecte nem mexa no dock durante o teste.
- O resultado esperado é **Completed without error**.

### Passo 5: Reler o SMART e comparar

```powershell
smartctl -a /dev/sdX -d sat
```

Compare com os valores anotados no Passo 3. Se realocados, pendentes ou não corrigíveis **aumentaram**, o disco está piorando.

### Passo 6: Varredura de superfície (opcional, só HDD)

Use o programa [**Victoria**](https://hdd.by/victoria/) em modo de **leitura** (sem remapear nem escrever). Muitos blocos lentos ou com erro indicam um disco em degradação. Em SSD, esta etapa é dispensável.

### Passo 7: Veredito

| Situação | Decisão |
|---|---|
| `PASSED`, autotestes sem erro, realocados/pendentes/não corrigíveis em 0 e estáveis | **Aprovado** |
| Poucos realocados, estáveis, e testes sem erro | **Uso com ressalvas** (dados sem importância, com backup) |
| `FAILED`, erro em algum autoteste, pendentes/não corrigíveis acima de 0 ou valores crescentes | **Reprovado** |

### Passo 8: Limpeza e formatação

Somente para discos aprovados. Veja [Limpeza, formatação e criação de partição](#limpeza-formatação-e-criação-de-partição).

### Passo 9: Validação de desempenho

Rode o **CrystalDiskMark** ([crystalmark.info](https://crystalmark.info/en/software/crystaldiskinfo/)) (botão "All") e compare com a especificação do fabricante. Pelo dock, o limite costuma ser o próprio dock ou a porta USB, e não o disco.

### Passo 10: Registro

Anote para cada disco: serial, modelo, horas ligado, realocados, vida útil (SSD), resultado dos testes e veredito.

## Windows (PowerShell / CMD)

Abra o PowerShell **como Administrador**.

### Instalar o smartmontools

O Windows não mostra todos os atributos SMART nativamente. Instale o [smartmontools](https://www.smartmontools.org/) pelo winget:

```powershell
winget install smartmontools.smartmontools
```

Feche e abra o PowerShell novamente e verifique a instalação:

```powershell
smartctl --version
smartctl --scan
```

Se o comando não for reconhecido, chame pelo caminho completo:

```powershell
& "C:\Program Files\smartmontools\bin\smartctl.exe" --scan
```

Alternativa gráfica: CrystalDiskInfo.

### Identificar o disco

```powershell
# Listar discos, partições e volumes
Get-Disk
Get-Partition
Get-Volume
```

```powershell
# Confirmar que o disco está em USB (troque o 2 pelo número do disco)
Get-Disk -Number 2 | Select-Object Number, FriendlyName, BusType, Size, PartitionStyle
```

### SMART com smartctl (dock/case USB)

O `/dev/sdX` segue a ordem do `Get-Disk`: disco 0 = `sda`, disco 1 = `sdb`, disco 2 = `sdc`, e assim por diante.

```powershell
# Informações do disco (modelo e serial reais)
smartctl -i /dev/sdc -d sat

# Todos os dados SMART
smartctl -a /dev/sdc -d sat
```

O `-d sat` (*SCSI/ATA Translation*) permite que o comando SMART atravesse o dock USB. Se der erro ou vier vazio, tente:

```powershell
smartctl -i /dev/sdc -d sat,12
smartctl -i /dev/sdc -d usbjmicron
smartctl -i /dev/sdc -d usbsunplus
```

Se nenhum funcionar, o dock não repassa o SMART. Use outro dock ou conecte o disco direto na placa-mãe.

#### Como interpretar o resultado

- `SMART overall-health self-assessment test result`: **PASSED** é o esperado. **FAILED** indica reprovação.
- `SSD_Life_Left`: vida útil restante em %. Quanto mais perto de 100, melhor.
- `Reallocated_Event_Count`: ideal **0**.
- `SATA_CRC_Error_Count` (ID 199): ideal **0**. Se crescer, suspeite do cabo ou do dock.

### Autotestes SMART

```powershell
# Teste curto (~2 min) e resultado
smartctl -t short /dev/sdc -d sat
smartctl -l selftest /dev/sdc -d sat

# Teste longo (pode levar horas) e resultado
smartctl -t long /dev/sdc -d sat
smartctl -l selftest /dev/sdc -d sat
```

### Saúde pelo Windows

```powershell
# Saúde e status operacional de todos os discos físicos
Get-PhysicalDisk | Select-Object DeviceId, FriendlyName, BusType, MediaType, HealthStatus, OperationalStatus, Size

# Ou apenas um disco (troque o 2 pelo número do disco)
Get-PhysicalDisk | Where-Object DeviceId -eq 2
```

```powershell
# Temperatura, desgaste, contadores de erro e horas ligado (somente leitura)
Get-PhysicalDisk | Get-StorageReliabilityCounter |
  Select-Object DeviceId, Temperature, Wear, ReadErrorsTotal, WriteErrorsTotal, PowerOnHours
```

> Em discos por USB, é normal o `MediaType` aparecer como **Unspecified** e o `Get-StorageReliabilityCounter` vir vazio. O `smartctl` é a fonte confiável.

### Verificação do sistema de arquivos (só se o disco tem letra)

Não se aplica a discos RAW. Troque `C:` pela letra do disco que está testando.

```powershell
# Varredura online, não bloqueia o volume (somente leitura)
Repair-Volume -DriveLetter C -Scan

# Mesma ideia usando chkdsk
chkdsk C: /scan

# Verificar se o volume está marcado como "sujo" (dirty)
fsutil dirty query C:
```

```powershell
# Corrigir erros (pode exigir reinicialização no disco do sistema)
chkdsk C: /f

# Corrigir erros E procurar setores defeituosos (lento, apenas HDD)
chkdsk C: /f /r
```

> ⚠️ Evite `chkdsk /r` em SSDs. É lento e desnecessário; `/f` basta.

## Como interpretar os dados SMART

### Atributos de HDD e SSD SATA

| ID  | Atributo                | O que significa                                                                                   |
|-----|-------------------------|---------------------------------------------------------------------------------------------------|
| 5   | Reallocated_Sector_Ct   | Setores defeituosos já remapeados. Valores crescentes são mau sinal.                              |
| 9   | Power_On_Hours          | Total de horas que o disco ficou ligado.                                                          |
| 12  | Power_Cycle_Count       | Quantas vezes o disco foi ligado e desligado.                                                     |
| 177 | Wear_Leveling_Count     | Desgaste do SSD (o significado varia por fabricante).                                             |
| 187 | Reported_Uncorrect      | Erros que não puderam ser corrigidos.                                                             |
| 194 | Temperature_Celsius     | Temperatura atual.                                                                                |
| 196 | Reallocated_Event_Count | **Muito importante.** Quantas vezes o disco moveu dados de uma área com falha para uma reserva. Zero é o ideal. |
| 197 | Current_Pending_Sector  | Setores aguardando remapeamento. Deve ser 0.                                                      |
| 198 | Offline_Uncorrectable   | Setores que falharam na varredura offline. Deve ser 0.                                            |
| 199 | UDMA_CRC_Error_Count    | Geralmente indica **cabo ou conexão ruim**, não o disco em si (alguns discos chamam de `SATA_CRC_Error_Count`). |
| 231 | SSD_Life_Left           | Vida útil restante do SSD, em %.                                                                  |
| 233 | Media_Wearout_Indicator | Vida útil restante do SSD (Intel e alguns outros).                                                |
| 234 | Flash_Writes_GiB        | Total de GiB gravados no disco.                                                                   |

> Os IDs e os nomes dos atributos variam entre fabricantes. Confie no **nome** e confira na saída do seu disco.

### Campos NVMe

| Campo                           | O que significa                                                    |
|---------------------------------|--------------------------------------------------------------------|
| Critical Warning                | Deve ser `0`.                                                      |
| Percentage Used                 | Vida útil do SSD consumida (100% = durabilidade nominal atingida). |
| Available Spare                 | Blocos reserva restantes. Deve estar bem acima do limite.          |
| Media and Data Integrity Errors | Deve ser `0`.                                                      |

## Sinais de alerta

- Saúde geral do SMART indica **FAILED**
- `Reallocated_Sector_Ct`, `Current_Pending_Sector` ou `Offline_Uncorrectable` acima de 0 e aumentando
- Erros de I/O repetidos no `dmesg` ou no log de eventos do Windows
- Leituras muito lentas, travamentos ou ruídos incomuns, como cliques (HDD)
- `Percentage Used` do SSD próximo de 100% ou `Available Spare` perto do limite

Se notar qualquer um desses sinais, **faça backup imediatamente** e planeje a substituição.

## Limpeza, formatação e criação de partição

> ⚠️ **Estes comandos apagam tudo e não têm volta.** Só use em discos aprovados no teste, e confirme o número do disco duas vezes. Errar o número pode apagar o disco do seu próprio computador.

```powershell
# 1. Confirme o número do disco: BusType deve ser USB e IsBoot/IsSystem devem ser False
Get-Disk | Select-Object Number, FriendlyName, BusType, Size, PartitionStyle, IsBoot, IsSystem
```

```powershell
# 2. Limpeza total: remove partições e o estado RAW (troque o 2 pelo número do disco)
# Se retornar o erro "The disk has not been initialized", pule para o passo 3.
Clear-Disk -Number 2 -RemoveData -RemoveOEM
```

```powershell
# 3. Inicializar o disco com tabela de partição GPT
Initialize-Disk -Number 2 -PartitionStyle GPT
```

```powershell
# 4. Criar a partição com todo o espaço, atribuir uma letra e formatar em exFAT
New-Partition -DiskNumber 2 -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem exFAT -NewFileSystemLabel "SSD_Externo"
```

```powershell
# 5. Conferir o resultado
Get-Volume
```

```powershell
# 6. Porem se o disco ja tiver partição e letra designada, voce pode só formatar:
Format-Volume -DriveLetter E -FileSystem exFAT -NewFileSystemLabel "HDD_Externo"
```

Observações:

- **exFAT** funciona bem em Windows, macOS e Linux. Para uso só no Windows, troque por `-FileSystem NTFS`.
- Em **HDD**, adicione `-Full` ao `Format-Volume` para uma formatação completa (mais lenta, mas grava em todo o disco e serve como teste extra). Em **SSD**, use a formatação rápida.
- Formatar não é uma sanitização segura. Se o disco tinha dados sensíveis, use uma ferramenta de apagamento seguro (*Secure Erase*) do fabricante.

## Integridade dos arquivos do Windows

Estes comandos verificam o **Windows instalado no seu PC**, e não o disco que está no dock.

```powershell
sfc /scannow
DISM /Online /Cleanup-Image /CheckHealth
DISM /Online /Cleanup-Image /ScanHealth
DISM /Online /Cleanup-Image /RestoreHealth
```

### Erros de disco no log de eventos

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='disk'} -MaxEvents 50
```

### Teste rápido de desempenho

```powershell
# Troque "c" pela letra do disco (só funciona em disco com letra)
winsat disk -drive c
```

## Linux (terminal)

### Instalar as ferramentas

```bash
# Debian / Ubuntu
sudo apt install smartmontools nvme-cli

# Fedora
sudo dnf install smartmontools nvme-cli
```

### Identificar os discos

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,MOUNTPOINT
sudo smartctl --scan
```

### SMART com smartctl

```bash
# Informações do dispositivo
sudo smartctl -i /dev/sdX

# Veredito geral de saúde (PASSED / FAILED)
sudo smartctl -H /dev/sdX

# Tabela de atributos
sudo smartctl -A /dev/sdX

# Tudo
sudo smartctl -a /dev/sdX
```

### Autotestes SMART

```bash
# Teste curto (~2 minutos)
sudo smartctl -t short /dev/sdX

# Teste longo (de minutos a horas, dependendo do tamanho do disco)
sudo smartctl -t long /dev/sdX

# Ver resultados depois que o teste terminar
sudo smartctl -l selftest /dev/sdX
```

### Discos NVMe

```bash
sudo smartctl -a /dev/nvme0
sudo nvme smart-log /dev/nvme0
```

### Verificação do sistema de arquivos

```bash
# A partição precisa estar DESMONTADA
sudo umount /dev/sdX1

# Apenas verificar, sem alterar nada (somente leitura)
sudo fsck -n /dev/sdX1

# Verificar e reparar
sudo fsck -f /dev/sdX1
```

### Setores defeituosos

```bash
# Teste somente leitura (seguro, modo padrão)
sudo badblocks -sv /dev/sdX
```

> ⚠️ Nunca use `badblocks -w` (teste destrutivo de escrita) em um disco com dados. Também não é recomendado para SSDs.

### Mensagens do kernel e erros de I/O

```bash
sudo dmesg | grep -i -E "error|i/o|ata|nvme|sector"
journalctl -k -p err
```

### Desempenho e uso

```bash
# Teste rápido de velocidade de leitura
sudo hdparm -Tt /dev/sdX

# Estatísticas de I/O em tempo real (pacote: sysstat)
iostat -x 2
```

## Licença

Este projeto está licenciado sob a [Licença MIT](LICENSE).
