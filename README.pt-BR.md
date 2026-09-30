<p align="right"><a href="README.md">🇺🇸 English</a></p>

# disk-health-cheatsheet

Referência rápida para verificar a saúde e a integridade de HDDs e SSDs usando comandos de terminal e PowerShell.

## Índice

- [Antes de começar](#antes-de-começar)
- [Windows (PowerShell / CMD)](#windows-powershell--cmd)
- [Linux (terminal)](#linux-terminal)
- [Como interpretar os dados SMART](#como-interpretar-os-dados-smart)
- [Sinais de alerta](#sinais-de-alerta)
- [Licença](#licença)

## Antes de começar

- A maioria dos comandos exige **administrador** (Windows) ou **root/sudo** (Linux).
- **Faça backup dos seus dados primeiro** se o disco já apresenta problemas. Ferramentas de reparo podem piorar um disco com defeito.
- Comandos marcados como *somente leitura* não alteram nada no disco.
- Substitua `/dev/sdX`, `/dev/nvme0` e `C:` pelo seu dispositivo ou letra de unidade.

## Windows (PowerShell / CMD)

Abra o PowerShell **como Administrador**.

## Instalando o smartmontools no PowerSheel pelo winget

```powershell
winget install smartmontools.smartmontools
```

```powershell
# verifique a instalacao
smartctl --version
smartctl --scan
```

### Visão geral da saúde

```powershell
# Listar discos, partições e volumes
Get-Disk
Get-Partition
Get-Volume
```

```powershell
# Se esta utilizando um docker ou case USB, Confirme que está em USB
# Altere o Number 2 pelo numero do disco.
Get-Disk -Number 2 | Select-Object Number, FriendlyName, BusType, Size, PartitionStyle
```

```powershell
# Comparar com a etiqueta do disco
smartctl --scan
# Troque o /dev/sdc, pelo nome do disco. sda para o disco 0, sdb para o 1, etc.
smartctl -i /dev/sdc -d sat
```

```powershell
# Saúde e status operacional de todos os discos físicos
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, HealthStatus, OperationalStatus, Size
# ou especifique o disco pelo numero (troqueo 2 pelo numero do disco)
Get-PhysicalDisk | Where-Object DeviceId -eq 2
```

```powershell
# Temperatura, desgaste, contadores de erro e horas ligado (somente leitura)
Get-PhysicalDisk | Get-StorageReliabilityCounter |
  Select-Object DeviceId, Temperature, Wear, ReadErrorsTotal, WriteErrorsTotal, PowerOnHours
```

### Verificação do sistema de arquivos

### "SMART with smartctl" e "SMART self-tests" no Windows

```
# 1. Informações do disco (confirmar modelo e serial reais)
smartctl -i /dev/sdc -d sat

# 2. Todos os dados SMART
smartctl -a /dev/sdc -d sat

# 3. Autoteste curto (~2 min) e resultado
smartctl -t short /dev/sdc -d sat
smartctl -l selftest /dev/sdc -d sat

# 4. Autoteste longo (pode levar horas), depois ver o resultado
smartctl -t long /dev/sdc -d sat
smartctl -l selftest /dev/sdc -d sat
```

### quando o disco possui letra/unidade
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

### Integridade dos arquivos do Windows

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
winsat disk -drive c
```

### SMART completo no Windows

O Windows não mostra todos os atributos SMART nativamente. Opções:

- Instalar o [smartmontools](https://www.smartmontools.org/) e usar os mesmos comandos `smartctl` da seção Linux.
- Usar uma ferramenta gráfica como o CrystalDiskInfo.

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

## Como interpretar os dados SMART

### Atributos de HDD e SSD SATA

| ID  | Atributo                 | O que significa                                                       |
|-----|--------------------------|-----------------------------------------------------------------------|
| 5   | Reallocated_Sector_Ct    | Setores defeituosos já remapeados. Valores crescentes são mau sinal.  |
| 9   | Power_On_Hours           | Total de horas que o disco ficou ligado.                              |
| 177 | Wear_Leveling_Count      | Desgaste do SSD (o significado varia por fabricante).                 |
| 187 | Reported_Uncorrect       | Erros que não puderam ser corrigidos.                                 |
| 194 | Temperature_Celsius      | Temperatura atual.                                                    |
| 197 | Current_Pending_Sector   | Setores aguardando remapeamento. Deve ser 0.                          |
| 198 | Offline_Uncorrectable    | Setores que falharam na varredura offline. Deve ser 0.                |
| 199 | UDMA_CRC_Error_Count     | Geralmente indica **cabo ou conexão ruim**, não o disco em si.        |
| 233 | Media_Wearout_Indicator  | Vida útil restante do SSD (Intel e alguns outros).                    |

### Campos NVMe

| Campo                           | O que significa                                           |
|---------------------------------|-----------------------------------------------------------|
| Critical Warning                | Deve ser `0`.                                             |
| Percentage Used                 | Vida útil do SSD consumida (100% = durabilidade nominal atingida). |
| Available Spare                 | Blocos reserva restantes. Deve estar bem acima do limite. |
| Media and Data Integrity Errors | Deve ser `0`.                                             |

## Sinais de alerta

- Saúde geral do SMART indica **FAILED**
- `Reallocated_Sector_Ct`, `Current_Pending_Sector` ou `Offline_Uncorrectable` acima de 0 e aumentando
- Erros de I/O repetidos no `dmesg` ou no log de eventos do Windows
- Leituras muito lentas, travamentos ou ruídos incomuns, como cliques (HDD)
- `Percentage Used` do SSD próximo de 100% ou `Available Spare` perto do limite

Se notar qualquer um desses sinais, **faça backup imediatamente** e planeje a substituição.

## Licença

Este projeto está licenciado sob a [Licença MIT](LICENSE).
