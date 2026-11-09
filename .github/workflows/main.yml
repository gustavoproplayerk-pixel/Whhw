name: RDP

on:
  workflow_dispatch:

jobs:
  secure-rdp:
    # Para máximo desempenho, se você tiver uma máquina self-hosted poderosa, use-a aqui.
    # Se usar o GitHub Hosted, ele já vem com recursos altos (8 vCPU, 32GB RAM).
    runs-on: windows-latest
    timeout-minutes: 3600

    steps:
      - name: Configure Core RDP Settings
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
                             -Name "fDenyTSConnections" -Value 0 -Force
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
                             -Name "UserAuthentication" -Value 0 -Force
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
                             -Name "SecurityLayer" -Value 0 -Force

          netsh advfirewall firewall delete rule name="RDP-Tailscale" 2>$null
          netsh advfirewall firewall add rule name="RDP-Tailscale" `
            dir=in action=allow protocol=TCP localport=3389

          Restart-Service -Name TermService -Force

      - name: Create RDP User with Secure Password
        run: |
          Add-Type -AssemblyName System.Security
          $charSet = @{
              Upper   = [char[]](65..90)
              Lower   = [char[]](97..122)
              Number  = [char[]](48..57)
              Special = ([char[]](33..47) + [char[]](58..64) +
                         [char[]](91..96) + [char[]](123..126))
          }
          $rawPassword = @()
          $rawPassword += $charSet.Upper | Get-Random -Count 4
          $rawPassword += $charSet.Lower | Get-Random -Count 4
          $rawPassword += $charSet.Number | Get-Random -Count 4
          $rawPassword += $charSet.Special | Get-Random -Count 4
          $password = -join ($rawPassword | Sort-Object { Get-Random })
          $securePass = ConvertTo-SecureString $password -AsPlainText -Force
          
          if (-not (Get-LocalUser -Name "RDP" -ErrorAction SilentlyContinue)) {
              New-LocalUser -Name "RDP" -Password $securePass -AccountNeverExpires
              Add-LocalGroupMember -Group "Administrators" -Member "RDP"
              Add-LocalGroupMember -Group "Remote Desktop Users" -Member "RDP"
          } else {
              Set-LocalUser -Name "RDP" -Password $securePass
          }
          
          echo "RDP_CREDS=User: RDP | Password: $password" >> $env:GITHUB_ENV
          
          if (-not (Get-LocalUser -Name "RDP")) {
              Write-Error "User creation/update failed"
              exit 1
          }

      - name: Install Tailscale
        run: |
          $tsUrl = "https://pkgs.tailscale.com/stable/tailscale-setup-1.82.0-amd64.msi"
          $installerPath = "$env:TEMP\tailscale.msi"
          
          Invoke-WebRequest -Uri $tsUrl -OutFile $installerPath
          Start-Process msiexec.exe -ArgumentList "/i", "`"$installerPath`"", "/quiet", "/norestart" -Wait
          Remove-Item $installerPath -Force

      - name: Establish Tailscale Connection
        run: |
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} --hostname=gh-runner-$env:GITHUB_RUN_ID
          
          $tsIP = $null
          $retries = 0
          while (-not $tsIP -and $retries -lt 10) {
              $tsIP = & "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4
              Start-Sleep -Seconds 5
              $retries++
          }
          
          if (-not $tsIP) {
              Write-Error "Tailscale IP not assigned. Exiting."
              exit 1
          }
          echo "TAILSCALE_IP=$tsIP" >> $env:GITHUB_ENV

      - name: Verify RDP Accessibility
        run: |
          Write-Host "RDP is ready on Tailscale IP: $env:TAILSCALE_IP"
          Write-Host "Use credentials from previous step to connect via RDP on port 3389"

      - name: Install Roblox Player
        run: |
          $robloxInstallerUrl = "https://www.roblox.com/download/client?os=win&platform=pc"
          $robloxInstallerPath = "$env:TEMP\RobloxPlayerLauncher.exe"
          
          Write-Host "Downloading Roblox Player..."
          Invoke-WebRequest -Uri $robloxInstallerUrl -OutFile $robloxInstallerPath
          
          Write-Host "Installing Roblox Player..."
          Start-Process -FilePath $robloxInstallerPath -ArgumentList "/S" -Wait
          
          Remove-Item $robloxInstallerPath -Force
          Write-Host "Roblox Player installed successfully."

      - name: Keep Runner Alive (Prevent Shutdown)
        run: |
          # Este passo mantém o runner ocupado indefinidamente (ou até o timeout de 3600 min)
          # Isso garante que o RDP continue disponível e o Roblox possa ser aberto manualmente 
          # ou via scripts adicionais sem que a VM seja desligada.
          Write-Host "Runner staying alive for up to 3600 minutes..."
          Start-Sleep -Seconds 216000 # 3600 minutos * 60 segundos
