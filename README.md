name: RDP

on:
  workflow_dispatch:

jobs:
  secure-rdp:
    runs-on: windows-latest
    timeout-minutes: 360

    steps:
      - name: Enable Secure RDP (NLA ON)
        run: |
          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
            -Name "fDenyTSConnections" -Value 0 -Force

          Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
            -Name "UserAuthentication" -Value 1 -Force

          Restart-Service TermService -Force

      - name: Create RDP User
        run: |
          $password = -join ((33..126) | Get-Random -Count 16 | % {[char]$_})
          $securePass = ConvertTo-SecureString $password -AsPlainText -Force

          New-LocalUser -Name "RDP" -Password $securePass -AccountNeverExpires
          Add-LocalGroupMember -Group "Administrators" -Member "RDP"
          Add-LocalGroupMember -Group "Remote Desktop Users" -Member "RDP"

          echo "RDP_PASSWORD=$password" >> $env:GITHUB_ENV

      - name: Install Tailscale
        run: |
          $url="https://pkgs.tailscale.com/stable/tailscale-setup-1.82.0-amd64.msi"
          $path="$env:TEMP\tailscale.msi"
          Invoke-WebRequest $url -OutFile $path
          Start-Process msiexec.exe -ArgumentList "/i `"$path`" /quiet /norestart" -Wait

      - name: Connect Tailscale
        run: |
          & "$env:ProgramFiles\Tailscale\tailscale.exe" up `
            --authkey=${{ secrets.TAILSCALE_AUTH_KEY }} `
            --hostname=gh-rdp-$env:GITHUB_RUN_ID

          $ip=& "$env:ProgramFiles\Tailscale\tailscale.exe" ip -4
          echo "TS_IP=$ip" >> $env:GITHUB_ENV

      - name: Info
        run: |
          Write-Host "======================"
          Write-Host "RDP ADDRESS : $env:TS_IP"
          Write-Host "USERNAME    : RDP"
          Write-Host "PASSWORD    : (saved in env)"
          Write-Host "TIME        : 6 HOURS"
          Write-Host "======================"

          Start-Sleep -Seconds 21600
